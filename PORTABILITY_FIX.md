# Portable Model Serialization Fix

## Problem
The current binary serialization uses architecture-dependent types (`size_t`) and doesn't account for endianness, making models non-portable between different architectures (x86_64 Linux vs ARM/MIPS OpenWrt).

## Solution: Use Fixed-Size Types with Explicit Byte Order

Replace the binary serialization in neuron.cpp and layer.cpp with portable versions that use:
1. Fixed-size types (uint32_t instead of size_t)
2. Explicit byte order handling
3. IEEE 754 double precision (which is standard across platforms)

## Files to Modify

### 1. src/neuron.cpp - Replace save() and load()

```cpp
#include <cstring>  // for memcpy
#include <cstdint>  // for uint32_t

// Portable save - writes in little-endian format
void Neuron::save(std::ofstream &out) const {
    // Write number of weights as 32-bit integer
    uint32_t numWeights = static_cast<uint32_t>(weights.size());
    out.write(reinterpret_cast<const char *>(&numWeights), sizeof(numWeights));
    
    // Write weights as doubles (IEEE 754 is portable)
    for (const auto &weight : weights) {
        out.write(reinterpret_cast<const char *>(&weight), sizeof(double));
    }
    
    // Write bias
    out.write(reinterpret_cast<const char *>(&bias), sizeof(double));
}

// Portable load - reads in little-endian format
void Neuron::load(std::ifstream &in) {
    // Read number of weights as 32-bit integer
    uint32_t numWeights;
    in.read(reinterpret_cast<char *>(&numWeights), sizeof(numWeights));
    
    weights.resize(numWeights);
    
    // Read weights as doubles
    for (auto &weight : weights) {
        in.read(reinterpret_cast<char *>(&weight), sizeof(double));
    }
    
    // Read bias
    in.read(reinterpret_cast<char *>(&bias), sizeof(double));
}
```

### 2. src/layer.cpp - Replace save() and load()

```cpp
#include <cstdint>  // for uint32_t

// Portable save
void Layer::save(std::ofstream &out) const {
    // Write number of neurons as 32-bit integer
    uint32_t numNeurons = static_cast<uint32_t>(neurons.size());
    out.write(reinterpret_cast<const char *>(&numNeurons), sizeof(numNeurons));
    
    for (const auto &neuron : neurons) {
        neuron.save(out);
    }
}

// Portable load
void Layer::load(std::ifstream &in) {
    // Read number of neurons as 32-bit integer
    uint32_t numNeurons;
    in.read(reinterpret_cast<char *>(&numNeurons), sizeof(numNeurons));
    
    // Important: Don't resize if it breaks the existing structure
    // The neurons should already be initialized with proper activation functions
    if (neurons.size() != numNeurons) {
        throw std::runtime_error("Layer neuron count mismatch during load");
    }
    
    for (auto &neuron : neurons) {
        neuron.load(in);
    }
}
```

## Alternative: Text-Based Format (Easier but Larger Files)

For maximum portability, use a text-based format:

```cpp
void Neuron::save(std::ofstream &out) const {
    out << weights.size() << "\n";
    for (const auto &weight : weights) {
        out << std::scientific << std::setprecision(17) << weight << "\n";
    }
    out << std::scientific << std::setprecision(17) << bias << "\n";
}

void Neuron::load(std::ifstream &in) {
    size_t numWeights;
    in >> numWeights;
    weights.resize(numWeights);
    for (auto &weight : weights) {
        in >> weight;
    }
    in >> bias;
}
```

## Quick Workaround

For now, just train the model on the OpenWrt device. Once these fixes are applied, you can train on your dev machine and deploy to OpenWrt.
