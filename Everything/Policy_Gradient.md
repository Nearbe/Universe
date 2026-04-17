# Policy Gradient

## Определение

Метод оптимизации параметров политики $\pi_\theta(a|s)$ напрямую через градиент ожидаемой награды.

## Теорема о градиенте политики

$$\nabla_\theta J(\theta) = \mathbb{E}_\tau\left[\sum_{t=0}^T \nabla_\theta \log \pi_\theta(a_t|s_t) \cdot R(\tau)\right]$$

где $\tau$ — траектория, $R(\tau)$ — суммарная награда.

## REINFORCE

Монте-Карло оценка градиента:
$$\theta \leftarrow \theta + \alpha \sum_t \nabla_\theta \log \pi_\theta(a_t|s_t) \cdot G_t$$

где $G_t = \sum_{k=t}^T \gamma^{k-t} r_k$.

## Actor-Critic

- **Actor:** $\pi_\theta(a|s)$ — политика
- **Critic:** $V_w(s)$ — оценка ценности

Обновление:
$$\nabla_\theta J \approx \nabla_\theta \log \pi_\theta(a_t|s_t) \cdot (r_t + \gamma V_w(s_{t+1}) - V_w(s_t))$$

Advantage $A = Q - V$ уменьшает дисперсию.

## PPO (Proximal Policy Optimization)

$$L^{CLIP}(\theta) = \mathbb{E}[\min(r_t(\theta) A_t, \text{clip}(r_t(\theta), 1-\varepsilon, 1+\varepsilon) A_t)]$$

Стабильнее vanilla policy gradient.
