# RLHF (Reinforcement Learning from Human Feedback)

## Определение

Метод обучения языковых моделей с использованием человеческих предпочтений через обучение с подкреплением.

## Этапы

### 1. SFT (Supervised Fine-Tuning)

Предобучение модели на высококачественных данных с учителем.

### 2. Обучение модели вознаграждения

- Сбор пар ответов $(y_1, y_2)$ для каждого запроса
- Человек выбирает предпочтительный
- Обучение reward model $r_\phi(x, y)$ через cross-entropy:
  $$\mathcal{L} = -\log \sigma(r_\phi(x, y_{\text{chosen}}) - r_\phi(x, y_{\text{rejected}}))$$

### 3. Оптимизация через PPO

$$\max_\theta \mathbb{E}[r_\phi(x, y_\theta)] - \beta \cdot D_{KL}(\pi_\theta \| \pi_{\text{SFT}})$$

KL-регуляризация предотвращает уход от начальной модели.

## Применения

ChatGPT, Claude, Gemini — все современные LLM используют RLHF.

## Проблемы

- Reward hacking: модель находит лазейки в reward model
- Дороговизна человеческой разметки
- Субъективность предпочтений
