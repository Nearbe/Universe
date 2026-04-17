# Diffusion models

## Определение

Генеративная модель, обучающая обращению процесса постепенного зашумления данных.

## Прямой процесс

Данные $x_0$ постепенно зашумляются гауссовым шумом:
$$q(x_t|x_{t-1}) = \mathcal{N}(x_t; \sqrt{1-\beta_t}x_{t-1}, \beta_t I)$$

При $t \to T$: $x_T \approx \mathcal{N}(0, I)$.

## Обратный процесс

Обучается сеть $\epsilon_\theta(x_t, t)$, предсказывающая шум:
$$p_\theta(x_{t-1}|x_t) = \mathcal{N}(x_{t-1}; \mu_\theta(x_t, t), \Sigma_\theta(x_t, t))$$

## Обучение

$$\mathcal{L} = \mathbb{E}_{t, x_0, \epsilon}\left[\|\epsilon - \epsilon_\theta(x_t, t)\|^2\right]$$

Простой MSE между истинным и предсказанным шумом.

## Сэмплирование

Из $x_T \sim \mathcal{N}(0, I)$ итеративно:
$$x_{t-1} = \frac{1}{\sqrt{\alpha_t}}\left(x_t - \frac{1-\alpha_t}{\sqrt{1-\bar{\alpha}_t}}\epsilon_\theta(x_t, t)\right) + \sigma_t z$$

## Применения

DALL-E 2, Imagen, Stable Diffusion — генерация изображений, аудио, молекул.

## Преимущества перед GAN

Стабильнее обучение, лучшее разнообразие, но медленнее сэмплирование.
