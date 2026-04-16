# STM32-NNUE Chess Evaluation Engine (EDD)

NNUE (*Efficiently Updatable Neural Network*) est une famille de réseaux conçus pour être **mis à jour incrémentalement** : au lieu de recalculer tout le réseau à chaque nouvel état, on met à jour une représentation interne en ne prenant en compte que ce qui a changé.  
Ce dépôt contient l’**Engineering Design Document (EDD)** d’un évaluateur NNUE **frugal** destiné à une cible **STM32F411 (Cortex-M4)**.

> But du dépôt : **présenter clairement le projet** (objectifs, contraintes, architecture de haut niveau, livrables) sans nécessiter la lecture préalable de l’EDD.

---

## Table des matières
- [Résumé](#résumé)
- [Pourquoi ce projet](#pourquoi-ce-projet)
- [Ce que fait le moteur](#ce-que-fait-le-moteur)
- [Contraintes & cible](#contraintes--cible)
- [Approche (haut niveau)](#approche-haut-niveau)
- [Architecture NNUE (vue produit)](#architecture-nnue-vue-produit)
- [Robustesse numérique](#robustesse-numérique)
- [Pipeline d’entraînement (vue d’ensemble)](#pipeline-dentraînement-vue-densemble)
- [Organisation du dépôt](#organisation-du-dépôt)
- [Construire l’EDD (PDF)](#construire-ledd-pdf)
- [Auteur](#auteur)

---

## Résumé

Ce projet implémente une **fonction d’évaluation** pour moteur d’échecs sur microcontrôleur, basée sur un **NNUE Tiny** exécuté **sans flottants** (arithmétique entière).  
La cible principale est **STM32F411** et la priorité est la **viabilité embarquée** : mémoire contrôlée, temps d’exécution déterministe, absence d’overflow.

---

## Pourquoi ce projet

Les évaluateurs “classiques” sur microcontrôleurs sont souvent basés sur des heuristiques manuelles (HCE). Ils sont rapides mais limités en expressivité.  
Le NNUE permet une évaluation **non-linéaire** (apprise) tout en restant **compatible CPU** et **incrémentale**, ce qui est essentiel pour tenir un débit de nœuds élevé sur MCU.

---

## Ce que fait le moteur

À chaque position d’échecs, le moteur doit produire un **score** (scalaire) indiquant “à quel point la position est favorable” pour un camp donné.  
Ce score sert ensuite à guider une recherche (Minimax/Negamax/Beam, etc.). Le projet se concentre sur **l’évaluateur NNUE**, pas sur un moteur complet.

---

## Contraintes & cible

- **MCU** : STM32F411 (ARM Cortex-M4 ~100 MHz)
- **RAM** : 128 Ko
- **Flash** : 512 Ko
- **Objectif mémoire modèle** : < ~50 Ko en Flash (ordre de grandeur)
- **Calcul** : uniquement **entiers** (pas de float), design compatible DSP (Cortex-M4)
- **Déterminisme** : aucune allocation dynamique pendant l’inférence

---

## Approche (haut niveau)

### 1) Représentation “sparse” (parcimonieuse)
La position est convertie en un vecteur **multi-hot** : plusieurs “1” actifs simultanément, mais peu par rapport à la taille totale.  
Dans cette version :
- **INPUT_SIZE = 768** correspond à : *(couleur × type de pièce × case)* = `2 × 6 × 64`.

Concrètement : chaque pièce active exactement **1 feature** (celle qui correspond à sa case, son type et sa couleur).  
Une position normale active donc ~**32 features** (au plus).

### 2) Mise à jour incrémentale
Quand une pièce bouge, seules quelques features changent (quelques “1” passent à 0 et quelques autres passent à 1).  
Le NNUE met à jour un **accumulateur** interne (couche cachée) en ajoutant/soustrayant uniquement les contributions qui ont changé.

### 3) Inférence entière et robuste
Les poids, biais, accumulateurs et opérations sont choisis pour :
- éviter tout overflow,
- rester rapides sur Cortex-M4,
- conserver une qualité d’évaluation acceptable (via distillation + QAT).

---

## Architecture NNUE (vue produit)

### Topologie
- Entrée : **768** features (multi-hot)
- Couche cachée : **32** neurones
- Sortie : **1** score

### Activation
- **CReLU (Clipped ReLU)** : on “clamp” l’activation dans un intervalle borné pour stabiliser les calculs entiers.

### Pourquoi “NNUE”
NNUE est adapté aux jeux / problèmes où l’état change peu entre deux pas (un coup d’échecs → peu de features modifiées).  
Le coût de mise à jour est donc **quasi constant** et ne dépend pas de `INPUT_SIZE` de façon dominante.

---

## Robustesse numérique

Le dépôt définit des **bornes** sur les poids/biais (format int16) et réalise les accumulations critiques en int32 lorsque nécessaire.  
L’EDD détaille :
- les bornes théoriques sur les accumulateurs,
- les bornes sur le produit scalaire de sortie,
- les choix de facteurs d’échelle (`QA`, `QB`, `SCALE`) afin d’éviter toute saturation.

> Note : l’objectif est une implémentation “safe by design”, pas seulement “ça marche en pratique”.

---

## Pipeline d’entraînement (vue d’ensemble)

L’entraînement vise à obtenir un NNUE compact mais “intelligent” :

1. **Génération des labels (Teacher)**  
   Un moteur fort (ex. Stockfish) produit des évaluations de positions.

2. **Distillation**  
   Le NNUE apprend à approximer ces évaluations (régression sur score).

3. **Quantization-Aware Training (QAT)**  
   Le modèle est entraîné en tenant compte des contraintes int16 (plages de valeurs, clipping, etc.) pour faciliter un déploiement sans flottants.

4. **Export binaire**  
   Les poids sont exportés dans un format directement intégrable côté C/C++ (lecture en bloc).

---

## Organisation du dépôt

- `tex_source/` : sources LaTeX de l’EDD (contenu principal, figures, démonstrations)
- `pdf_export/` : PDF compilé de l’EDD (si présent)
- `assets/` : ressources (schémas, poids, visuels)


---
**Auteur** : Ajyad HASSANI  
*Élève-Ingénieur à Télécom SudParis*