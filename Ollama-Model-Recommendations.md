## ✅ Recommended Ollama Model (Low VRAM Setup)

To keep the entire stack within **~9GB of VRAM**, I recommend the following model and settings:

### 💡 Daily Driver Model

- **Model**: [`TheAzazel/gemma3-4b-abliterated`](https://ollama.com/TheAzazel/gemma3-4b-abliterated)  
- **Base**: [Gemma 3](https://ollama.com/library/gemma3) (4-bit quantized version)

This is a highly capable model for **general-purpose voice interaction**. While not ideal for coding tasks, it performs **very well for conversations, memory recall, and daisy-chained prompts**.

I've tested it extensively with both:

- 🧠 [Persistent Memory](./AI-Persistent-Memory.md)
- 🔁 [Vocal Daisy Chaining](./Vocal-Command-Daisy-Chaining.md)

### 🧠 Recommended Settings (in Home Assistant)

| Setting         | Value      |
|----------------|------------|
| Context Window | `8000`     |
| Max Messages   | `35`       |
| Keep Alive     | `-1`       |

With these settings, the model runs smoothly **alongside the entire Docker Compose stack** under ~9GB VRAM.

---

### ⚠️ Important Note on Safety

This is an "**abliterated**" model — meaning it lacks built-in safety filters or refusal logic. It **will not** refuse unsafe or off-topic prompts.

If you require a model with stronger safeguards, use the **official quantized Gemma 3 model** instead:

- 👉 [`gemma3:4b`](https://ollama.com/library/gemma3:4b)

---

### 📉 VRAM Usage Tip

The **context window** size has the biggest impact on VRAM consumption. While `gemma3` supports up to **128K context**, setting it that high can spike VRAM usage to **15+GB**.

> ✅ **Recommended sweet spot**:  
> `8000` tokens — fits well on most consumer GPUs.

---

### 🖼️ Screenshots

| Shot 1 | Shot 2 |
|----------------------------------|----------------------|
| ![UI1 Screenshot](https://i.imgur.com/dek3wav.jpeg) | ![UI2 Screenshot](https://i.imgur.com/qClrM7I.jpeg) |

