---
title: Building an Algebraic Hierarchy
separator: <!--s-->
verticalSeparator: <!--v-->
theme: solarized
highlight-theme: monokai
css: custom.css
revealOptions:
  controls: false
  slideNumber: false
  transition: 'slide'
  backgroundTransition: 'fade'
---


# Building an Algebraic Hierarchy

1. Basics
2. Morphisms
3. Subobjects

---

## Basics

```lean
class Inv (α : Type) where
  /-- The inversion function -/
  inv : α → α
```

---

## Morphisms

---

## Subobjects

