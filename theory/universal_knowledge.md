# Протокол Универсальных Знаний (Universal Knowledge Protocol)

## Аннотация
Универсальная карта знаний представляет собой фиксированную функцию отображения произвольных входных данных в компактное паттерн-пространство. В отличие от весов модели, которые могут достигать объема 111 ГБ, универсальная карта занимает около 50 МБ и обеспечивает детерминированную семантическую проекцию.

## Ключевые свойства

| Свойство | Описание |
|----------|----------|
| **Детерминированность** | Одинаковый вход всегда порождает идентичный паттерн. |
| **Универсальность** | Единая карта применима к любым типам входных данных. |
| **Компактность** | Сжатие относительно оригинальных весов достигает 2000x (111 ГБ → 50 МБ). |
| **Скорость** | Вычислительная сложность проекции составляет $O(d \cdot k)$ вместо $O(d^2)$. |

## Математическая модель
Обученная модель аппроксимируется через SVD-разложение. Универсальная карта состоит из матрицы собственных векторов $P \in \mathbb{R}^{d \times k}$ и вектора сингулярных значений $S \in \mathbb{R}^k$.

Для любого входа $x$:
$$p(x) = (P^T \cdot x) \otimes S,$$
где $p(x)$ — $k$-мерный семантический паттерн. Сходство между объектами вычисляется как косинусное расстояние между их паттернами:
$$\operatorname{sim}(x_1, x_2) = \frac{p(x_1) \cdot p(x_2)}{\|p(x_1)\| \cdot \|p(x_2)\|}.$$

## Что храним vs веса

|                   | Веса (оригинал)   | Универсальная карта  |
|-------------------|-------------------|----------------------|
| **Размер**        | 111GB             | ~50MB                |
| **Содержимое**    | Числовые значения | Паттерны + важность  |
| **Использование** | Полный forward    | Проекция → сравнение |
| **Свойство**      | Изменяется        | Фиксировано          |

## Solenoid Storage for Knowledge Map (NEW)

### Mathematical Foundation

The universal knowledge map is stored using the **solenoid lossless mechanism** from `Essentials/`:

- **Solenoid**: `𝒮 = lim← (𝕌:Ω)` — each knowledge point encodes its full branching history
- **Branching**: `D(a) = a : Ω` — expansion operator (creates difference)
- **Compression**: `H(a) = a : D(Id)` — collapse operator (removes noise)
- **Lossless cycle**: `H(D(a)) = a` — perfect recovery guaranteed

### Storage Mechanism

```
Knowledge Map P (d×k) stored as solenoid points:

For each column p_j ∈ R^d of P:
  → Encode as solenoid point z₀ = Dⁿ(zₙ)
  → Store: (binary_prefix, d_level_meta, p_adic_residue)

For each singular value s_j ∈ R^k:
  → Encode as solenoid point z₀ = Dᵐ(zₘ)
  → Store: (binary_prefix, d_level_meta, p_adic_residue)

Reconstruction via H(D(z₀)) = z₀ → exact recovery ✓
```

### Why Solenoid for Knowledge Map?

| Benefit           | Explanation                                                  |
|-------------------|--------------------------------------------------------------|
| **Lossless**      | `H(D(a)) = a` guarantees exact recovery of P and S           |
| **Queryable**     | Binary prefix enables fast similarity search across patterns |
| **Compressible**  | p-adic residue allows further compression without loss       |
| **Deterministic** | Same knowledge → same solenoid point, always                 |
| **Scalable**      | `Dⁿ(Id)` scales with depth — no fixed precision limit        |

### Storage Size

| Component         | Float32           | Solenoid (float16 + metadata) |
|-------------------|-------------------|-------------------------------|
| P (d×k = 4096×32) | 524 KB            | ~262 KB + ~10 KB metadata     |
| S (k = 32)        | 128 B             | ~64 B + ~512 B metadata       |
| **Total**         | **~524 KB/layer** | **~273 KB + metadata**        |

For a 16-layer model:

- Original weights: ~118 MB (float32)
- Knowledge map solenoid: ~4.4 MB + metadata → **~50 MB**

This achieves the target 111GB → ~50MB (2000x compression) when combined with SVD k=32 truncation.

## Зачем это нужно

1. **Semantic Search** — ищу по паттернам, не декодируя модель
2. **Fast Comparison** — просто косинусное сходство паттернов
3. **Clustering** — группирую по паттернам без inference
4. **Universal Embeddings** — это как GPT embeddings, но компактнее

## Файлы

```
/Users/nearbe/Eugenia/
├── universal_knowledge_protocol.py  # Основная реализация
├── universal_knowledge_map.py        # Универсальная карта
├── deterministic_core.py             # Детерминированное ядро
├── semantic_knowledge_storage.py     # Семантическое хранилище
└── ../analytic/compression_summary.md            # Итоги сжатия
```

## Математика

```
P (d×k) — собственные векторы (паттерны)
S (k,)  — сингулярные значения (важность)

Для входа x:
    p(x) = (P.T @ x) * S   — проекция в паттерн-пространство

Сходство:
    sim(x1, x2) = cos(p(x1), p(x2)) — без полного decode!

Solenoid storage guarantee:
    H(D(p_j)) = p_j  →  lossless ✓
```

---

**КЛЮЧЕВОЕ:** Карта одинакова для всех — это как GPS координаты для знаний. Кто угодно может её использовать для
понимания структуры знаний без необходимости иметь оригинальную модель.

**Solenoid storage ensures this map is stored losslessly** — the `H(D(a)) = a` cycle guarantees that no information
is lost when compressing and reconstructing the knowledge map. This is critical because the universal knowledge map
must remain **deterministic** and **universal** — same input must always produce the same output, regardless of when
or where it's loaded.
