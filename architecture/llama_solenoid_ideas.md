# Технологические Принципы llama.cpp в Контексте Solenoid-сжатия

## Аннотация
Настоящий отчет содержит анализ методов квантования и сжатия, применяемых в проекте `llama.cpp`, и предложения по их адаптации для архитектуры Solenoid в рамках проекта «Евгения». Основное внимание уделено многомасштабной иерархии и гибридным методам разложения.

## 1. Многомасштабная иерархия (Multi-scale Hierarchy)

**Концепция:** Использование форматов K-quant (Q2_K — Q8_K) предполагает наличие глобального масштаба $d$ и локальных коэффициентов $scales[j]$ для каждого подблока.

**Предложение:** Перейти от фиксированного параметра $Id$ к иерархической структуре масштабов:
1. **Уровень 0:** Глобальный масштаб (аналог $d$ в Q8_K).
2. **Уровень 1:** Блочный масштаб (аналог $scales[j]$ в Q4_K).
3. **Уровень 2:** Субблочный масштаб (высокая точность).

**Математическая модель:**
Для суперблока $B$ размером $QK\_K = 256$:
- Глобальный масштаб: $scale_{global} = d_{super}$.
- Для каждого подблока $j$ (16 элементов):
  $$scale_{block} = d_{super} \cdot scales[j],$$
  $$x_{recon}[j] = scale_{block} \cdot \operatorname{quantize}(x[j]).$$

**Преимущества:** Адаптивность к локальной гетерогенности весов. Однородные блоки используют минимальное число уровней, гетерогенные — детальную иерархию.

---

## 2. Importance-weighted квантование (AWQ)

**Откуда:** IQ форматы (IQ2_XXS, IQ3_XXS) — принимают `imatrix` (importance matrix per element).

**Идея:** Weight-aware квантование — элементы с большим влиянием на output получают больше бит.

**Применение в Solenoid:**
```python
def quantize_solenoid_aware(data: np.ndarray, importance: np.ndarray):
    """
    importance: np.ndarray — вес влияния каждого элемента (например, из SVD)
    
    Элементы с большим importance:
      - Больше итераций Solenoid (Dⁿ с большим n)
      - Меньше сжатия, больше точности
    
    Элементы с малым importance:
      - Меньше итераций
      - Сильнее сжимаются
    """
    # Adaptive depth per element
    n_iter = np.clip(importance * max_depth / importance.max(), min_depth, max_depth).astype(int)
    
    result = np.zeros_like(data)
    for i in range(len(data)):
        result[i] = encode_solenoid(data[i], n_iter[i])
    
    return result
```

**Почему лучше:** Рациональное распределение бит. Не все веса одинаково важны — attention q/k/v projections важнее, чем output projection.

---

## 3. Hybrid SVD + K-quant

**Идея:** Комбинировать глобальную низкоранговую аппроксимацию с блочным квантованием residual.

**Pipeline:**
```
1. SVD: W ≈ Uₖ × Σₖ × Vₖᵀ          (низкоранговое ядро)
2. Residual: R = W - UₖΣₖVₖᵀ         (что осталось)
3. K-quant на residual: quantize_K(R) (блочное квантование)
4. Storage: (Uₖ, Σₖ) + k-quant(R)
```

**Почему это работает:**
- SVD убирает **глобальную структуру** (основные направления вариации)
- Residual содержит **локальные детали**, которые хорошо квантуются блоками
- K-quant на residual эффективнее, чем на оригинал — residual менее гетерогенный

**Размер:**
```
Original:    d × d × 4B
SVD (k):     2 × d × k × 4B + k² × 4B
Hybrid:      2 × d × k × 4B + k² × 4B + (d²/QK_K) × block_size_k_quant(R)

При k=32, d=4096, Q4_K на residual:
  SVD:         2 × 4096 × 32 × 4 + 32² × 4 = 1.05 MB
  Residual:    (4096²/256) × 236 ≈ 6.15 MB (если residual dense)
  Но residual sparse → можно + mask → ещё меньше
```

**Оптимизация:** Если residual sparse (много нулей после SVD), хранить только non-zero блоки.

---

## 4. Adaptive block size

**Откуда:** llama.cpp — фиксированный QK_K=256 для всех K-quant.

**Идея:** Разный размер блока для разных подматриц.

```
Большой блок (256): однородные веса (мало overhead)
Малый блок (16): гетерогенные веса (много параметров на блок, но точность)

Автоматический выбор per-submatrix:
  Для каждой подматрицы:
    variance = std(weights_submatrix)
    if variance < threshold:
      block_size = 256  # однородный → большой блок
    else:
      block_size = 16   # гетерогенный → малый блок
```

**Применение в Solenoid:**
```python
def adaptive_block_size(matrix: np.ndarray, threshold=0.1):
    """Определяет оптимальный размер блока для каждого подблока."""
    blocks = split_into_blocks(matrix, max_block_size=256)
    results = []
    
    for block in blocks:
        variance = np.std(block)
        if variance < threshold:
            block_size = 256
            n_iter = deep_level  # глубокое сжатие
        else:
            block_size = 16
            n_iter = shallow_level  # мелкая итерация
    
        results.append(encode_solenoid(block, n_iter, block_size))
    
    return concatenate_blocks(results)
```

**Почему лучше:** Не тратим параметры на scale там, где они не нужны (однородные блоки).

---

## 5. Non-linear Solenoid activation

**Откуда:** llama.cpp — только линейная реконструкция `x_recon = scale × Q + bias`.

**Идея:** Заменить линейную реконструкцию на нелинейную, обученную на residual.

```python
def nonlinear_reconstruct(quantized: np.ndarray, scales: np.ndarray, bias: np.ndarray):
    """
    Вместо линейного: x_recon = scale × Q + bias
    
    Нелинейный Solenoid:
      x_recon = Solenoid(Q, scale, bias, iteration_depth)
    
    Где Solenoid — нелинейная функция:
      S(x) = x + α × tanh(β × (x - μ)) + γ × (x - μ)²
    
    Параметры α, β, γ обучаются на residual от линейной реконструкции.
    """
    base = scales * quantized + bias
    
    # Non-linear correction
    residual = quantized / scales  # нормализованный residual
    correction = alpha * np.tanh(beta * (residual - mu)) + gamma * (residual - mu) ** 2
    
    return base + correction
```

**Почему лучше:**
- Линейная аппроксимация llama.cpp не учитывает нелинейные паттерны в весах
- SVD residual часто имеет нелинейную структуру (хвосты, выбросы)
- Non-linear activation может снизить ошибку при том же bpw

**Обучение параметров:**
```
1. Линейная реконструкция → residual R_lin
2. Оптимизировать α, β, γ: min ||R_lin - nonlin_correction||²
3. Фиксировать параметры для всего тензора (или per-block)
```

---

## Сводная таблица приоритетов

| # | Идея | Сложность | Потенциал | Зависимости |
|---|------|-----------|-----------|-------------|
| 1 | Multi-scale иерархия | Низкая | Высокий | Только добавление уровня масштабов |
| 2 | Importance-weighted | Средняя | Высокий | Нужен imatrix (из SVD или калибровки) |
| 3 | Hybrid SVD + K-quant | Высокий | Очень высокий | Требует переработки pipeline |
| 4 | Adaptive block size | Средняя | Средний | Нужен анализ гетерогенности |
| 5 | Non-linear activation | Высокий | Средний | Нужен dataset для обучения |

---

## Рекомендация по внедрению

**Приоритет P0:** Multi-scale иерархия (идея 1)
- Минимальный риск, максимальная отдача
- Интегрируется в текущую Solenoid структуру
- Прямая аналогия с Q4_K/Q6_K из llama.cpp

**Приоритет P1:** Hybrid SVD + K-quant (идея 3)
- Самый большой потенциал
- Требует переработки `../architecture/eugenia_architecture.md` pipeline
- Позволяет достичь bpw < 0.1 для 111GB → 50MB

**Приоритет P2:** Importance-weighted (идея 2)
- Работает как надстройка над P0 или P1
- imatrix можно взять из SVD (сингулярные значения)
- Быстрая реализация
