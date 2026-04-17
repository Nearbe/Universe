# Transformer

## Определение

Архитектура нейронной сети, основанная на механизме внимания (attention), без рекуррентности и свёрток.

## Компоненты

**Multi-Head Attention:**
$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$$

**Positional Encoding:**
Синусоидальное кодирование позиции:
$$PE_{(pos, 2i)} = \sin(pos/10000^{2i/d})$$

**Feed-Forward:**
$$FFN(x) = \max(0, xW_1 + b_1)W_2 + b_2$$

**Layer Normalization + Residual:**
$$\text{LayerNorm}(x + \text{Sublayer}(x))$$

## Архитектура

- **Encoder:** N слоёв (Multi-Head Attention → FFN → Norm)
- **Decoder:** N слоёв (Masked Attention → Cross Attention → FFN → Norm)

## Применения

- NLP: BERT, GPT, T5
- Computer Vision: Vision Transformer (ViT)
- Мультимодальные модели: DALL-E, CLIP

## Сложность

Attention: $O(n^2 \cdot d)$ для последовательности длины $n$.
