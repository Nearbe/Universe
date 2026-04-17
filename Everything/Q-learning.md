# Q-learning

## Определение

Алгоритм обучения без модели (model-free), оценивающий функцию $Q(s, a)$.

## Обновление

$$Q(s_t, a_t) \leftarrow Q(s_t, a_t) + \alpha \left[ r_{t+1} + \gamma \max_a Q(s_{t+1}, a) - Q(s_t, a_t) \right]$$

где $\alpha$ — скорость обучения, $\gamma$ — дисконт.

## Свойства

- При бесконечном посещении всех $(s, a)$ и убывающем $\alpha$: $Q(s, a) \to Q^*(s, a)$
- Off-policy: обучает оптимальную политику, следуя $\epsilon$-жадной

## Глубокий Q-learning (DQN)

Вместо таблицы — нейросеть $Q_\theta(s, a)$:
- **Experience replay:** запоминание и повторное использование переходов
- **Target network:** отдельная сеть для вычисления цели
- **Loss:** MSE между $Q_\theta(s,a)$ и $r + \gamma \max_{a'} Q_{\theta^-}(s', a')$

## Применения

Atari (DeepMind), робототехника, управление ресурсами, игры (AlphaGo — в комбинации с MCTS).

## Проблемы

- Переоценка (overestimation bias) → Double DQN
- Нестабильность обучения → Dueling DQN, Rainbow
