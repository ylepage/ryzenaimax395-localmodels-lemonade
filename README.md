# ryzenaimax395-localmodels-lemonade
# 🍋 Lemonade sur AMD Ryzen AI+ 395 Max — Guide de sélection de modèles LLM locaux

> **Configuration matérielle :** AMD Ryzen AI+ 395 Max (Strix Halo), 16 cœurs Zen 5, iGPU RDNA 3.5 (gfx1151), NPU XDNA2 (~50 TOPS), **128 Go RAM DDR5**
>
> **Objectif :** Identifier les meilleurs modèles LLM pour une configuration Lemonade locale, en tenant compte des backend NPU/GPU/CPU, de la quantisation, et de la taille de fenêtre de contexte.

---

## ⚠️ Avertissement — Les résultats varient

Les performances rapportées ici sont des **estimations basées sur l'architecture matérielle et les benchmarks publics**. La réalité dépend de :

- **Votre charge de travail** : un prompt de 50 tokens n'utilise pas la RAM de la même façon qu'un document de 128 000 tokens.
- **Le contexte actif** : la fenêtre de contexte chargée impacte directement la RAM et les tokens/seconde (voir [section contexte](#contexte-et-r%C3%A9am%C3%A9nagement)).
- **Les mises à jour** : Lemonade, llama.cpp et ROCm évoluent rapidement — une version récente peut changer la donne.
- **Les modèles concurrents** : de nouveaux modèles sortent chaque semaine ; ce guide est un instantané.

**Vérifiez toujours avec `lemonade backends` et vos propres tests avant de conclure.**

---

## 1. Le Matériel — Ce que vous avez

| Composant | Détail | Impact pour l'inférence |
|-----------|--------|------------------------|
| **CPU** | Zen 5, 16 cœurs | CPU pur possible, lent mais fiable |
| **NPU** | XDNA2 (~50 TOPS) | FLM / ryzenai-llm — inférence ultra-efficace, faible consommation |
| **iGPU** | RDNA 3.5, 16 CUs (gfx1151) | ROCm via vLLM (expérimental) / Vulkan via llama.cpp |
| **RAM** | 128 Go DDR5 | **Le vrai avantage** — vous pouvez faire tenir des modèles 70B en Q4 |

### Stratégie d'accélération par taille de modèle

| Taille du modèle | Backend recommandé | Pourquoi |
|------------------|--------------------|----------|
| ≤ 7B | NPU (FLM / ryzenai-llm) | Décharge CPU/GPU, faible énergie, très rapide |
| 7B–35B | Vulkan (llama.cpp) | Support stable sur Strix Halo |
| 35B–70B | Vulkan + RAM overflow | 128 Go tient un 70B Q4 (~42 Go) |
| 70B+ | Vulkan + Q4_K_M | MoE permet de garder le modèle actif plus petit |
| Vitesse max | vLLM ROCm (Linux) | Meilleures performances, mais expérimental sur gfx1151 |

---

## 2. Top 5 — Modèles natifs Lemonade (`lemonade pull`)

### 🥇 Qwen3.6-35B-A3B-MTP-GGUF

```bash
lemonade pull Qwen3.6-35B-A3B-MTP-GGUF
```

| Critère | Détail |
|---------|-------|
| Architecture | MoE (Mixture of Experts) — 35B params, ~3B actifs par token |
| MTP (Multi-Token Prediction) | ✅ Génère plusieurs tokens en une seule passe — **2× plus rapide** |
| Backend idéal | Vulkan (llama.cpp) |
| Taille disque | ~22 Go (Q8_0) |
| RAM en contexte court (4k) | ~25 Go |
| RAM en contexte long (32k+) | ~40 Go |
| Points forts | Vitesse d'inférence exceptionnelle grâce à MoE + MTP. Excellent en raisonnement, code, et conversation. |
| Limites | MTP moins efficace en Q4 — visez Q8_0 sur 128 Go |
| **Cas d'usage** | **Modèle généraliste principal — le meilleur compromis perf/qualité sur cette config** |

---

### 🥈 Qwen3-Coder-30B-A3B-Instruct-GGUF

```bash
lemonade pull Qwen3-Coder-30B-A3B-Instruct-GGUF
```

| Critère | Détail |
|---------|-------|
| Architecture | MoE — 30B params, ~3B actifs |
| Spécialité | **Code, tool-calling, raisonnement** |
| Backend idéal | Vulkan |
| Taille disque | ~20 Go (Q8_0) |
| RAM en contexte court (4k) | ~23 Go |
| RAM en contexte long (32k+) | ~38 Go |
| Points forts | Entraîné sur code et tool-calling. Label "hot" dans Lemonade — la communauté l'utilise activement. |
| Limites | Pas de MTP — légèrement plus lent que Qwen3.6-35B en débit |
| **Cas d'usage** | **Développement, agent IA, automation (Claude Code, OpenHands, etc.)** |

---

### 🥉 Gemma-4-31B-it-MTP-GGUF

```bash
lemonade pull Gemma-4-31B-it-MTP-GGUF
```

| Critère | Détail |
|---------|-------|
| Architecture | Dense — 31B params |
| MTP | ✅ Multi-Token Prediction |
| Multimodal | ✅ Vision intégrée (image + texte) |
| Backend idéal | Vulkan / ROCm |
| Taille disque | ~20 Go (Q8_0) |
| RAM en contexte court (4k) | ~23 Go |
| RAM en contexte long (32k+) | ~38 Go |
| Points forts | Le plus polyvalent de sa famille — vision, chat, tool-calling, MTP. Google a poussé fort sur Gemma 4. |
| Limites | Dense — pas de MoE, donc 31B actifs par token (vs 3B pour Qwen3.6-35B) |
| **Cas d'usage** | **Chat multimodal — analyse d'images + conversation en français** |

---

### 4️⃣ Qwen3.5-122B-A10B-MTP-GGUF

```bash
lemonade pull Qwen3.5-122B-A10B-MTP-GGUF
```

| Critère | Détail |
|---------|-------|
| Architecture | MoE — 122B params, ~10B actifs |
| MTP | ✅ |
| Backend idéal | Vulkan + RAM overflow |
| Taille disque | ~60 Go (Q4_K_M) |
| RAM en contexte court (4k) | ~65 Go |
| RAM en contexte long (32k+) | ~85 Go |
| Points forts | Le plus grand modèle qui rentre dans votre config. Qualité proche des modèles cloud 400B+. |
| Limites | Latence plus élevée malgré MoE. MTP partiellement efficace en Q4. Q4 impacte la qualité — visez Q6_K si la perf le permet |
| **Cas d'usage** | **Recherche, rédaction longue, analyse de documents — quand la qualité prime sur la vitesse** |

---

### 5️⃣ Qwen3-8B-GGUF

```bash
lemonade pull Qwen3-8B-GGUF
```

| Critère | Détail |
|---------|-------|
| Architecture | Dense — 8B params |
| Backend idéal | **NPU (FLM / ryzenai-llm)** — parfaitement dans la taille cible |
| Taille disque | ~5 Go (Q8_0) |
| RAM en contexte court (4k) | ~6 Go |
| RAM en contexte long (32k+) | ~10 Go |
| Points forts | Ultra-rapide sur NPU. Zéro charge sur le CPU. Qualité surprenante pour sa taille. |
| Limites | Capacité de raisonnement limitée vs modèles 30B+ |
| **Cas d'usage** | **Tâches rapides : tri d'email, résumé court, chat léger — tout ce qui doit tourner en arrière-plan** |

---

## 3. Top 3 — Modèles HuggingFace (via `lemonade pull --hf`)

### 🥇 bartowski/DeepSeek-R1-Distill-Llama-70B-GGUF

```bash
lemonade pull --hf bartowski/DeepSeek-R1-Distill-Llama-70B-GGUF
```

| Critère | Détail |
|---------|-------|
| Architecture | Dense — 70B params |
| Spécialité | **Raisonnement profond (chain-of-thought)** |
| Variantes GGUF | Q4_K_M (~42 Go), Q6_K (~55 Go), Q8_0 (~70 Go) |
| Backend idéal | Vulkan + RAM |
| RAM en contexte court (4k) | 45–73 Go |
| RAM en contexte long (32k+) | 60–90 Go |
| Points forts | Meilleur modèle open pour le raisonnement analytique. Distillé depuis DeepSeek-R1. 77 500+ téléchargements. |
| Limites | Dense 70B — latence importante en Q4. En Q8_0, la RAM système sera saturée |
| **Cas d'usage** | **Mathématiques, logique, raisonnement juridique, analyse de contrats — tout ce qui demande du raisonnement long** |

---

### 🥈 MaziyarPanahi/Llama-3.3-70B-Instruct-GGUF

```bash
lemonade pull --hf MaziyarPanahi/Llama-3.3-70B-Instruct-GGUF
```

| Critère | Détail |
|---------|-------|
| Architecture | Dense — 70B params |
| Spécialité | **Généraliste, conversation, création de contenu** |
| Variantes GGUF | Q4_K_M, Q5_K_M, Q8_0 |
| Backend idéal | Vulkan |
| RAM en contexte court (4k) | 45–73 Go |
| RAM en contexte long (32k+) | 60–90 Go |
| Points forts | 90 800+ téléchargements. Llama 3.3 est le généraliste par excellence — excellent en français, conversation longue, rédaction. |
| Limites | Dense 70B, latence. Moins performant en raisonnement pur que DeepSeek-R1 distill |n| **Cas d'usage** | **Rédaction longue, écrivain de fiction, conversation en français, analyse littéraire** |

---

### 🥉 ggml-org/SmolVLM2-2.2B-Instruct-GGUF

```bash
lemonade pull --hf ggml-org/SmolVLM2-2.2B-Instruct-GGUF
```

| Critère | Détail |
|---------|-------|
| Architecture | Dense — 2.2B params |
| Spécialité | **Vision + vidéo (multimodal)** |
| Backend idéal | **NPU (FLM)** — parfaitement dans la taille cible |
| Taille disque | ~1.5 Go (Q8_0) |
| RAM en contexte court (4k) | ~2 Go |
| RAM en contexte long (16k) | ~3 Go |
| Points forts | Vision et vidéo dans un modèle de 2.2B. 14 290+ téléchargements. Ultra-léger. |
| Limites | 2.2B = capacité de raisonnement faible ; c'est un modèle de perception visuelle |
| **Cas d'usage** | **Analyse d'images, classification visuelle, extraction de texte depuis des captures d'écran — tâches visuelles légères** |

---

## 4. Contexte et réaménagement — Le paramètre caché

La fenêtre de contexte est le paramètre le plus sous-estimé quand on parle de performance locale. **Elle détermine directement combien de RAM est consommée et à quelle vitesse le modèle génère.**

### Comment le contexte impacte la RAM et les tokens/seconde

Chaque token de contexte (entrée) doit être chargé en mémoire pour le calcul d'attention. Pour un modèle de 35B en Q8_0 :

| Contexte | RAM estimée (Qwen3.6-35B) | tokens/s estimés |
|----------|--------------------------|-----------------|
| 2 048 (court) | ~25 Go | 50-60 |
| 8 192 (moyen) | ~30 Go | 45-55 |
| 32 768 (long) | ~40 Go | 35-45 |
| 128 000 (max) | ~55 Go | 25-35 |

**Règle empirique :** chaque ×2 de contexte ajoute ~10-15 % de RAM et réduit les t/s d'environ 10-15 %. Les modèles MoE y gagnent car le contexte ne charge que les experts actifs.

### Stratégies d'optimisation du contexte

#### 1. `n_ctx` adapté à la tâche (méthode principale)

Ne chargez **jamais** un contexte de 128k si votre tâche n'en nécessite que 8k. Dans Lemonade, le contexte est souvent géré par les paramètres du modèle intégré ou via la requête API :

```bash
# Pour une conversation normale — contexte court, max vitesse
lemonade run Qwen3.6-35B-A3B-MTP-GGUF --n_ctx 8192

# Pour analyser un document de 50 000 tokens — contexte long
lemonade run Qwen3.6-35B-A3B-MTP-GGUF --n_ctx 65536
```

#### 2. `n_ctx` dynamique avec swap (pour les modèles 70B+)

Pour les grands modèles en contexte long, le swap RAM→disque sauve la config mais tue les performances :

```bash
# 128 Go de RAM — swap activé si besoin, sur disque SSD NVMe
lemonade run DeepSeek-R1-Distill-Llama-70B-GGUF --n_ctx 32768 --gpu_layers 99 --mlock 0
```

**Règle :** ne jamais activer le swap pour des modèles ≤ 35B sur 128 Go — la RAM suffit.

#### 3. Offloading partiel (GPU + RAM CPU)

Sur Strix Halo, divisez les couches entre GPU (Vulkan) et RAM CPU pour garder le modèle rapide tout en gardant de la RAM libre pour le contexte :

```bash
# 35B MoE — couche sur GPU, contexte en RAM
lemonade run Qwen3.6-35B-A3B-MTP-GGUF --gpu_layers 99 --n_ctx 32768 --batch_size 512
```

#### 4. Fenêtre glissante (sliding window attention)

Si votre modèle le supporte (Qwen3, Gemma 4), la fenêtre glissante ne charge que les N derniers tokens en contexte plein et le reste en contexte « rapide » :

```bash
# 32k de contexte plein + 128k « visible » en attention réduite
lemonade run Qwen3.6-35B-A3B-MTP-GGUF --n_ctx 131072 --rwkv_window 32768
```

#### 5. Préréchauffage (prefill) du contexte

Pour un gros prompt (ex: un document), préchargez le contexte avant de demander de la génération :

```bash
# API : chargez le contexte en premier
curl http://localhost:1337/v1/chat/completions \
  -d '{"model": "Qwen3.6-35B-A3B-MTP-GGUF", "max_tokens": 1, "messages": [{"role": "user", "content": "<document de 50k tokens>"}]}'

# Puis générez — le contexte est déjà en RAM
```

#### 6. Matrice de contexte par modèle

| Modèle | Contexte recommandé par défaut | Contexte max sans swap | Contexte max avec swap |
|--------|-------------------------------|------------------------|------------------------|
| Qwen3-8B | 8 192 | 65 536 | 128 000 |
| Qwen3-Coder-30B-A3B | 16 384 | 32 768 | 65 536 |
| Qwen3.6-35B-A3B-MTP | 16 384 | 32 768 | 65 536 |
| Gemma-4-31B-it-MTP | 16 384 | 32 768 | 65 536 |
| Qwen3.5-122B-A10B-MTP | 8 192 | 16 384 | 32 768 |
| DeepSeek-R1-70B (Q4) | 8 192 | 16 384 | 32 768 |
| Llama-3.3-70B (Q4) | 8 192 | 16 384 | 32 768 |
| SmolVLM2-2.2B | 4 096 | 16 384 | 32 768 |

---

## 5. Tableau comparatif — Lequel utiliser quand ?

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      MATRICE DE SÉLECTION DE MODÈLE                          │
├──────────────────────┬───────────────┬───────────────┬──────────────────────┤
│          CAS          │   VITESSE     │   QUALITÉ     │   MODÈLE CHOISI      │
│                      │   (priorité)  │   (priorité)  │                      │
├──────────────────────┼───────────────┼───────────────┼──────────────────────┤
│ Chat quotidien       │   Qwen3-8B    │   Gemma-4-31B │   Qwen3.6-35B        │
│ (FR, rapide)         │   NPU         │   + MTP       │   + MTP              │
├──────────────────────┼───────────────┼───────────────┼──────────────────────┤
│ Code / Dev / Agents  │   Qwen3-8B    │   Llama-3.3-70B│  Qwen3-Coder         │
│                      │   NPU         │                 │  30B-A3B             │
├──────────────────────┼───────────────┼───────────────┼──────────────────────┤
│ Raisonnement profond │   Qwen3-8B    │   DeepSeek-R1 │   DeepSeek-R1        │
│ (math, logique)      │   NPU         │   70B Q6_K    │   70B Q6_K           │
├──────────────────────┼───────────────┼───────────────┼──────────────────────┤
│ Analyse d'images     │   SmolVLM2    │   Gemma-4-31B │   Gemma-4-31B        │
│ + chat               │   NPU         │   MTP (vision)│   MTP (vision)       │
├──────────────────────┼───────────────┼───────────────┼──────────────────────┤
│ Rédaction longue     │   Qwen3-8B    │   Qwen3.5-122B│   Qwen3.5-122B       │
│ (articles, essai)    │   NPU         │   MoE Q4      │   MoE Q6_K           │
├──────────────────────┼───────────────┼───────────────┼──────────────────────┤
│ Tâches en arrière-   │   Qwen3-8B    │   —           │   Qwen3-8B           │
│ plan (tri, résumé)   │   NPU         │               │   (FLM/NPU)          │
└──────────────────────┴───────────────┴───────────────┴──────────────────────┘
```

---

## 6. Configuration recommandée

### Matricielle d'accélération

```
┌──────────────────────────────────────────────────────────────┐
│                   STRIX HALO BACKEND MATRIX                  │
├────────────────┬──────────┬──────────┬──────────────────────┤
│ Modèle         │ Backend  │ Quant.   │ Estimation perf.     │
├────────────────┼──────────┼──────────┼──────────────────────┤
│ Qwen3-8B       │ FLM/NPU  │ Q8_0     │ ⚡⚡⚡⚡⚡ 150+ t/s     │
│ Qwen3.6-35B    │ Vulkan   │ Q8_0     │ ⚡⚡⚡⚡   40-60 t/s    │
│ Qwen3-Coder-30B│ Vulkan   │ Q8_0     │ ⚡⚡⚡⚡   35-50 t/s    │
│ Gemma-4-31B    │ Vulkan   │ Q8_0     │ ⚡⚡⚡⚡   35-55 t/s    │
│ DeepSeek-R1-70B│ Vulkan   │ Q6_K     │ ⚡⚡⚡    15-25 t/s    │
│ Llama-3.3-70B  │ Vulkan   │ Q4_K_M   │ ⚡⚡⚡    10-20 t/s    │
│ Qwen3.5-122B   │ Vulkan   │ Q4_K_M   │ ⚡⚡⚡    12-20 t/s    │
└────────────────┴──────────┴──────────┴──────────────────────┘

> Les tokens/s sont estimés — vérifiez avec vos propres tests. Le Strix Halo (gfx1151) a un support ROCm/vLLM expérimental ; Vulkan via llama.cpp est plus stable.
```

### Règle d'or — Quantisation sur 128 Go

| Modèle | Max params | Quantisation | RAM utilisée (ctx court) |
|--------|-----------|-------------|--------------------------|
| 8B | 8B | **Q8_0** | ~6 Go |
| 30-35B | 35B | **Q8_0** | ~25 Go |
| 70B | 70B | **Q6_K** | ~58 Go |
| 122B MoE | 122B (10B actifs) | **Q4_K_M** | ~65 Go |

Avec 128 Go, **vous pouvez avoir 2-3 modèles chargés simultanément** — ex: Qwen3-8B (NPU) + Qwen3.6-35B (Vulkan) en même temps.

### Workflow recommandé

1. **Qwen3-8B sur NPU** → tâches légères, tri, résumé, chat rapide
2. **Qwen3.6-35B-A3B-MTP sur Vulkan** → modèle principal, qualité/vitesse optimale
3. **DeepSeek-R1-70B sur Vulkan** → chargé à la demande pour le raisonnement complexe
4. **Gemma-4-31B-MTP** → quand la vision multimodale est nécessaire

---

## 7. Notes de compatibilité Lemonade

- **NPU** : les modèles ≤ 7B rentrent naturellement. Les modèles MoE 30B+ ne sont **pas** supportés sur NPU actuellement.
- **Vulkan** : support stable sur gfx1151 via llama.cpp. C'est le backend recommandé pour les modèles 7B–35B.
- **vLLM ROCm** : expérimental sur Strix Halo sous Linux. Prometteur mais pas encore stable en production.
- **Cloud offload** : Lemonade peut router vers OpenAI/OpenRouter/Fireworks en complément — utile quand un modèle local n'est pas assez puissant.
- **Modèles HuggingFace** : utilisez `lemonade pull --hf <repo>/<model>` pour ajouter n'importe quel GGUF à votre serveur.

---

> **Dernier mot :** ce guide reflète l'état de Lemonade v10.10 et les modèles disponibles en juillet 2026. Les résultats varient selon l'usage, la version de vos pilotes, et les mises à jour. Testez, itérez, et ajustez `n_ctx` et la quantisation en fonction de vos besoins réels. 🍋
