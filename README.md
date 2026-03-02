# STM32-NNUE Chess Evaluation Engine

Ce dépôt présente la conception d'une fonction d'évaluation neuronale de type **NNUE** (*Efficiently Updatable Neural Network*) optimisée pour l'architecture **ARM Cortex-M4** (STM32F411). 

L'objectif est de démontrer la viabilité d'un modèle d'IA non-linéaire sous des contraintes matérielles strictes (128 Ko RAM / 512 Ko Flash) en s'affranchissant totalement du calcul en virgule flottante.

## 🛠️ Spécifications de la Cible
* **Processeur** : STM32F411CEU6 @100MHz (Cortex-M4).
* **Contrainte CPU** : Inférence sans accélération matérielle (No Tensor Unit).
* **Contrainte Mémoire** : Empreinte Flash du modèle < 50 Ko (9.6% de la Flash totale).
* **Arithmétique** : Quantification intégrale en **Int16** via *Quantization-Aware Training* (QAT).



## 🧠 Architecture & Optimisation

### Inférence Incrémentale (Complexité $O(32)$)
Le goulot d'étranglement des réseaux de neurones sur microcontrôleur est le produit matriciel. Ce design exploite la parcimonie du **One-Hot Encoding** (768 entrées) pour transformer l'inférence de la couche cachée en une mise à jour différentielle de l'accumulateur :

$$A_{t+1} = A_{t} - w_{\text{source}} + w_{\text{dest}}$$

Chaque mouvement de pièce ne nécessite que **64 opérations d'addition/soustraction entières**, permettant au moteur de maintenir un débit de nœuds (NPS) élevé malgré la fréquence d'horloge réduite.



### Modélisation du Réseau
* **Topologie** : 768 (Entrée) $\rightarrow$ 32 (Couche cachée) $\rightarrow$ 1 (Sortie).
* **Activation** : CReLu (Clipped ReLU) pour assurer la non-linéarité sans overflow.
* **Distillation** : Modèle entraîné à partir de labels générés par Stockfish pour capturer une évaluation positionnelle complexe.

## 📂 Organisation du Dépôt

* **`assets/`** : Ressources graphiques, schémas TikZ et poids du réseau quantifié.
* **`pdf_export/`** : Version compilée de l'Engineering Design Document (EDD).
* **`tex_source/`** : Code source LaTeX du document technique, incluant les démonstrations mathématiques de l'incrémentalité.

---
**Auteur** : Ajyad HASSANI  
*Élève-Ingénieur à Télécom SudParis*