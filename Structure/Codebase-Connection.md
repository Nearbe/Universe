# (Δ,Q) Formalization ↔ Codebase Connection

## Overview

This document connects the (Δ,Q) formalization to the existing codebase (HodonNumber.swift, FormulaRegistry.swift).

---

## HodonNumber.swift Connection

### Parameters

From `Core/HodonNumber.swift`:

```swift
/// Параметр Δ = ln|Re/Im| — рассинхронизация потоков
public var delta: Double

/// Частота вибрации: f = e^(-Δ)
var frequency: Double {
    exp(-delta)
}

/// Период: T = 1/f = e^Δ
var period: Double {
    exp(delta)
}

/// Фрактальная размерность: D_f = 2 + Δ
var fractalDimension: Double {
    2.0 + delta
}
```

### (Δ,Q) Mapping

| HodonNumber Property | (Δ,Q) Parameter | Formula |
|---------------------|-----------------|---------|
| delta | Δ | ln|Re/Im| |
| frequency | f | e^(-Δ) |
| period | T | e^Δ |
| fractalDimension | D_f | 2 + Δ |
| amplitude | |Ψ| | √(Re² + Im²) |
| phase | φ | atan2(Im, Re) |

### Q Parameter

**From Physics/Вибрации:**
```
Q = e^|Δ| (vibration energy)
```

**In HodonNumber:**
```
Q = exp(abs(delta))
```

**Not yet in codebase** — need to add.

---

## FormulaRegistry.swift Connection

### Existing Formulas

From `Core/FormulaRegistry.swift`:

```swift
// Δ = ln(Re/Im)
public static func delta(re: Double, im: Double) -> FormulaResult<Double>

// f = f₀ × exp(-Δ)
public static func frequency(delta: Double, f0: Double) -> FormulaResult<Double>

// T = e^Δ
public static func period(delta: Double) -> FormulaResult<Double>

// Q = e^Δ (structure complexity)
public static func structureComplexity(delta: Double) -> FormulaResult<Double>

// D_f = 2 + Δ
public static func fractalDimensionTopological(delta: Double) -> FormulaResult<Double>
```

### (Δ,Q) Mapping

| FormulaRegistry Function | (Δ,Q) Parameter | Formula |
|-------------------------|-----------------|---------|
| delta() | Δ | ln(Re/im) |
| frequency() | f | f₀ × e^(-Δ) |
| period() | T | e^Δ |
| structureComplexity() | Q | e^Δ |
| fractalDimensionTopological() | D_f | 2 + Δ |

### Note on Q

**FormulaRegistry uses:** Q = e^Δ (structure complexity)
**Physics/Вибрации uses:** Q = e^|Δ| (vibration energy)

**Difference:** Absolute value in exponent.

**Resolution:** Both are valid:
- Q = e^Δ: signed complexity (can be < 1)
- Q = e^|Δ|: unsigned energy (always ≥ 1)

---

## New Formulas to Add

### 1. Vibration Energy (Q)

```swift
/// Энергия вибрации: Q = e^|Δ|
public static func vibrationEnergy(delta: Double) -> FormulaResult<Double> {
    let sig = FormulaSignature(id: .vibrationEnergy, category: .delta, name: "Энергия вибрации",
                               formula: "Q = e^|Δ|",
                               definition: "Энергия вибрации ходона.",
                               links: [.delta, .frequency, .mass])
    return FormulaResult(value: exp(abs(delta)), signature: sig)
}
```

### 2. Mass from (Δ,Q)

```swift
/// Масса через (Δ,Q): m ∝ e^(Δ/2) · f(Q)
public static func massFromDQ(delta: Double, q: Double) -> FormulaResult<Double> {
    let sig = FormulaSignature(id: .massFromDQ, category: .nuclear, name: "Масса через (Δ,Q)",
                               formula: "m ∝ e^(Δ/2) · f(Q)",
                               definition: "Масса частицы через параметры (Δ,Q).",
                               links: [.delta, .vibrationEnergy, .mass])
    let value = PhysicalConstants.protonMass * exp(delta / 2) * exp(0.30 * q)
    return FormulaResult(value: value, signature: sig)
}
```

### 3. Synchronization with Q

```swift
/// Синхронизация с Q: S = 1 - σ(Δ) · 0.05 · (1 - e^(-Q))
public static func synchronizationWithQ(sigmaDelta: Double, q: Double) -> FormulaResult<Double> {
    let sig = FormulaSignature(id: .synchronizationWithQ, category: .synchronization, name: "Синхронизация с Q",
                               formula: "S = 1 - σ(Δ) · 0.05 · (1 - e^(-Q))",
                               definition: "Синхронизация с учетом энергии вибрации Q.",
                               links: [.synchronization, .vibrationEnergy])
    let value = 1.0 - sigmaDelta * 0.05 * (1.0 - exp(-q))
    return FormulaResult(value: value, signature: sig)
}
```

### 4. Blood Operator

```swift
/// Оператор крови: B(Δ) = IDFT(1/(n²+1) · DFT(Δ))
public static func bloodOperator(delta: [Double]) -> FormulaResult<[Double]> {
    let sig = FormulaSignature(id: .bloodOperator, category: .synchronization, name: "Оператор крови",
                               formula: "B(Δ) = IDFT(1/(n²+1) · DFT(Δ))",
                               definition: "Оператор крови — фильтр низких частот.",
                               links: [.synchronization, .fourierTransform])
    // Implementation: DFT → low-pass filter → IDFT
    let n = delta.count
    var result = [Double](repeating: 0, count: n)
    for k in 0..<n {
        var sum = 0.0
        for j in 0..<n {
            let angle = -2.0 * Double.pi * Double(k * j) / Double(n)
            sum += delta[j] * cos(angle)
        }
        result[k] = sum / Double(n) / (Double(k * k) + 1.0)
    }
    return FormulaResult(value: result, signature: sig)
}
```

### 5. Eye Operator

```swift
/// Оператор глаз: G(Δ) = -Δ
public static func eyeOperator(delta: Double) -> FormulaResult<Double> {
    let sig = FormulaSignature(id: .eyeOperator, category: .synchronization, name: "Оператор глаз",
                               formula: "G(Δ) = -Δ",
                               definition: "Оператор глаз — инверсия Re/Im.",
                               links: [.synchronization, .delta])
    return FormulaResult(value: -delta, signature: sig)
}
```

---

## Simulation Connections

### SynchronizationSimulation.swift

From `Core/Math/SynchronizationSimulation.swift`:

**Current:** Monte Carlo simulation of 889 hodons
**Enhancement:** Add (Δ,Q) parameters

```swift
// Add to simulation
let q = exp(abs(delta))  // Vibration energy
let mass = massFromDQ(delta: delta, q: q)  // Mass from (Δ,Q)
let syncWithQ = synchronizationWithQ(sigmaDelta: sigma, q: q)  // Sync with Q
```

### BloodEyeLoopSimulation.swift

From `Core/Math/BloodEyeLoopSimulation.swift`:

**Current:** Blood-eye loop convergence
**Enhancement:** Add (Δ,Q) tracking

```swift
// Add to simulation
let q = exp(abs(delta))  // Vibration energy at each step
let convergence = q → 1.0  // Q converges to 1
```

---

## Summary

### Existing Code

- **HodonNumber.swift:** Δ, frequency, period, fractalDimension
- **FormulaRegistry.swift:** delta(), frequency(), period(), structureComplexity()

### Missing from Code

- **Q parameter:** vibrationEnergy() = e^|Δ|
- **Mass from (Δ,Q):** massFromDQ() = e^(Δ/2) · f(Q)
- **Synchronization with Q:** synchronizationWithQ() = 1 - σ(Δ)·0.05·(1-e^(-Q))
- **Blood operator:** bloodOperator() = IDFT(1/(n²+1)·DFT(Δ))
- **Eye operator:** eyeOperator() = -Δ

### Next Steps

1. **Add new formulas** to FormulaRegistry.swift
2. **Update HodonNumber.swift** with Q parameter
3. **Enhance simulations** with (Δ,Q) tracking
4. **Create unified theory document**
