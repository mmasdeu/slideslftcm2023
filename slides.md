---
title: Building an Algebraic Hierarchy
separator: <!--s-->
verticalSeparator: <!--v-->
theme: simple
highlight-theme: zenburn
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

---

### Data-carrying classes

We have already seen data-carrying classes:

class One (α : Type) where
  /-- The element one -/
  one : α

What the `class` command does:

1. Defines a structure `One`:
  - with parameter `α : Type`, and
  - a single field `one : α`.
2. Marks it as a class: instance resolution.
3. `One α` appears as instance-implicit in its own fields:

```lean
#check One.one -- lftcm.One.one {α : Type} [self : One₁ α] : α

example (α : Type) [One α] : α := One.one

@[class] structure One' (α : Type) where
  /-- The element one -/
  one : α

#check One'.one -- lftcm.One'.one {α : Type} (self : One' α) : α

---

### Adding notation

We will use `𝟙` to avoid confusion with the usual `1`.

```lean
@[inherit_doc]
notation "𝟙" => One.one

example {α : Type} [One α] : α := 𝟙

example {α : Type} [One α] : (𝟙 : α) = 𝟙 := rfl
```

---

### Operation

We will use a "diamond" to avoid confusion with `*`.

```lean
class Dia (α : Type) where
  dia : α → α → α

infixl:70 " ⋄ "   => Dia.dia

class SemigroupDia (α : Type) where
  toDia : Dia α
  /-- Diamond is associative -/
  dia_assoc : ∀ a b c : α, a ⋄ b ⋄ c = a ⋄ (b ⋄ c)
```

Note that in `dia_assoc` we can use `⋄`, since `toDia : Dia α` is in the local context. But this fails:

```lean
example {α : Type} [SemigroupDia α] (a b : α) : α := a ⋄ b -- fails
```

We can fix this by adding the ``instance`` attribute later.

```lean
attribute [instance] SemigroupDia.toDia

example {α : Type} [SemigroupDia α] (a b : α) : α := a ⋄ b  -- works
```

---

### A better way

Avoid doing all this manual work (writing `toDia`, the `attribute [instance]` statement...

- Instead, use `extends`!

```lean
class SemigroupDia' (α : Type) extends Dia α where
  /-- Diamond is associative -/
  dia_assoc : ∀ a b c : α, a ⋄ b ⋄ c = a ⋄ (b ⋄ c)

example {α : Type} [SemigroupDia' α] (a b : α) : α := a ⋄ b -- works!
```

---

### Adding axioms

```lean
class DiaOneClass (α : Type) extends One α, Dia α where
  /-- One is a left neutral element for diamond. -/
  one_dia : ∀ a : α, 𝟙 ⋄ a = a
  /-- One is a right neutral element for diamond -/
  dia_one : ∀ a : α, a ⋄ 𝟙 = a
```

---

### Monoids

We could do now:
```
class MonoidDiaBad (α : Type) where
  toSemigroupDia : Semigroup α
  toDiaOneClass : DiaOneClass α
```

But then we get two completely unrelated diamond operations
`MonoidDiaBad.toSemigroupDia.toDia.dia` and `MonoidDiaBad.toDiaOneClass.toDia.dia`.

Instead, use `extends`:

```lean
class MonoidDia (α : Type) extends SemigroupDia α, DiaOneClass α
```
This does some magic:
```lean
example {α : Type} [MonoidDia α] :
  (MonoidDia.toSemigroupDia.toDia.dia : α → α → α) = MonoidDia.toDiaOneClass.toDia.dia := rfl
```

---

So the `class` command did some magic for us. Compare:

```lean
#check MonoidDiaBad.mk
-- lftcm.MonoidDiaBad.mk {α : Type} (toSemigroupDia : Semigroup α) (toDiaOneClass : DiaOneClass α) : MonoidDiaBad α

#check MonoidDia.mk
-- lftcm.MonoidDia.mk {α : Type} [toSemigroupDia : SemigroupDia α] [toOne : One α] (one_dia : ∀ (a : α), 𝟙 ⋄ a = a)
  (dia_one : ∀ (a : α), a ⋄ 𝟙 = a) : MonoidDia α
```

It also auto-generated an instance ``Monoid.toDiaOneClass``
which is *not* a field but has the expected signature.

---

### Groups

We define class for an inverse function, and notation for it.
```lean
class Inv (α : Type) where
  /-- The inversion function -/
  inv : α → α

@[inherit_doc]
postfix:max "⁻¹" => Inv.inv
```
This allows already to define a group:
```lean
class GroupDia (G : Type) extends MonoidDia G, Inv G where
  inv_dia : ∀ a : G, a⁻¹ ⋄ a = 𝟙
```

In the exercises you will prove that this is enough (the other direction follows).

---


## Rings

- We hard-coded the notation `⋄` for multiplication, now we need two operations!
- We will just prove multiplicative lemmas, generate automatically the additive versions.

```lean
class AddSemigroup (α : Type) extends Add α where
/-- Addition is associative -/
  add_assoc : ∀ a b c : α, a + b + c = a + (b + c)

@[to_additive]
class Semigroup (α : Type) extends Mul α where
/-- Multiplication is associative -/
  mul_assoc : ∀ a b c : α, a * b * c = a * (b * c)
```

---

```lean
class AddMonoid (α : Type) extends AddSemigroup α, AddZeroClass α

@[to_additive]
class Monoid (α : Type) extends Semigroup α, MulOneClass α

attribute [to_additive existing] Monoid.toMulOneClass

@[to_additive]
lemma left_inv_eq_right_inv' {M : Type} [Monoid M] {a b c : M} (hba : b * a = 1) (hac : a * c = 1) : b = c := by
  rw [← one_mul c, ← hba, mul_assoc, hac, mul_one b]
```

---


```lean
class Neg (α : Type) where
  /-- The negation function -/
  neg : α → α

@[inherit_doc]
prefix:max "-" => Neg.neg

class AddCommSemigroup (α : Type) extends AddSemigroup α where
  add_comm : ∀ a b : α, a + b = b + a

@[to_additive]
class CommSemigroup (α : Type) extends Semigroup α where
  mul_comm : ∀ a b : α, a * b = b * a
```

---

```lean
class AddCommMonoid (α : Type) extends AddMonoid α, AddCommSemigroup α

@[to_additive]
class CommMonoid (α : Type) extends Monoid α, CommSemigroup α

class AddGroup (G : Type) extends AddMonoid G, Neg G where
  neg_add : ∀ a : G, -a + a = 0

@[to_additive]
class Group (G : Type) extends Monoid G, Inv G where
  mul_left_inv : ∀ a : G, a⁻¹ * a = 1

class AddCommGroup (G : Type) extends AddGroup G, AddCommMonoid G

@[to_additive]
class CommGroup (G : Type) extends Group G, CommMonoid G
```

---

### Rings, finally

Finally we can define rings:

```lean
class Ring (R : Type) extends AddGroup R, Monoid R, MulZeroClass R where
  /-- Multiplication is left distributive over addition -/
  left_distrib : ∀ a b c : R, a * (b + c) = a * b + a * c
  /-- Multiplication is right distributive over addition -/
  right_distrib : ∀ a b c : R, (a + b) * c = a * c + b * c

instance {R : Type} [Ring R] : AddCommGroup R :=
{ Ring.toAddGroup with
  add_comm := by sorry
}
```

---

### The integers form ring

```
instance : Ring ℤ where
  add := (· + ·)
  add_assoc := _root_.add_assoc
  zero := 0
  zero_add := by simp
  add_zero := by simp
  neg := (-(·))
  neg_add := by simp
  mul := (· * ·)
  mul_assoc := _root_.mul_assoc
  one := 1
  one_mul := by simp
  mul_one := by simp
  zero_mul := by simp
  mul_zero := by simp
  left_distrib := Int.mul_add
  right_distrib := Int.add_mul
```

---


### Modules over rings

These involve several types:  commutative additive groups
equipped with a scalar multiplication by elements of some ring.

```lean
class SMul (α : Type) (β : Type) where
  /-- Scalar multiplication -/
  smul : α → β → β

infixr:73 " • " => SMul.smul
```

```lean
class Module (R : Type) [Ring R] (M : Type) [AddCommGroup M] extends SMul R M where
  zero_smul : ∀ m : M, (0 : R) • m = 0
  one_smul : ∀ m : M, (1 : R) • m = m
  mul_smul : ∀ (a b : R) (m : M), (a * b) • m = a • b • m
  add_smul : ∀ (a b : R) (m : M), (a + b) • m = a • m + b • m
  smul_add : ∀ (a : R) (m n : M), a • (m + n) = a • m + a • n
```

Note that `Smul R M` is in `extends`, while `AddCommGroup M` is not!

- This is so that the class inference system stays sane. Otherwise, it would keep looking for some type `R` with `Ring R`.
- No problem with `Smul R M` since it mentions both `R` and `M`.

**Rule:**  each class appearing in the
`extends` clause should mention every type appearing in the parameters.

---

### Examples

First, a ring is a module over itself.

```lean
instance selfModule (R : Type) [Ring R] : Module R R where
  smul := fun r s ↦ r*s
  zero_smul := zero_mul
  one_smul := one_mul
  mul_smul := mul_assoc
  add_smul := Ring.right_distrib
  smul_add := Ring.left_distrib
```

An abelian group is a ℤ-module.

```lean
def nsmul [Zero M] [Add M] : ℕ → M → M
  | 0, _ => 0
  | n + 1, a => a + nsmul n a

def zsmul {M : Type} [Zero M] [Add M] [Neg M] : ℤ → M → M
  | Int.ofNat n, a => nsmul n a
  | Int.negSucc n, a => -(nsmul n.succ a)

instance abGrpModule (A : Type) [AddCommGroup A] : Module ℤ A where
  smul := zsmul
  zero_smul := by simp [zsmul, nsmul]
  one_smul := by simp [zsmul, nsmul]
  mul_smul := sorry
  add_smul := sorry
  smul_add := sorry
```

### Problem!!

We have two module structures for the ring ℤ over ℤ itself.
1. `abGrpModule ℤ`, since ℤ is a abelian group, and
2. `selfModule ℤ`, since ℤ is a ring.
The fact that these two coincide is a theorem, not true by definition! This is a bad diamond.

- Not all diamonds are bad: a diamond having a ``Prop``-valued class at the bottom
cannot be bad since any too proofs of the same statement are definitionally equal.
- The one we have now is bad: ``smul`` is data, not a proof, and we have two constructions that are not definitionally equal.

To fix this:  make sure that going from a rich structure to a
poor structure is always done by forgetting data, not by defining data.

### Solution

We can modify the definition of `AddMonoid` to include a `nsmul` data
field and some ``Prop``-valued fields ensuring this operation is provably the one we constructed
above.

- Can be given default values using `:=` after their type in the definition below.
- Most instances would be constructed exactly as with our previous
definitions.
- In the special case of ℤ we will be able to provide specific values.

```lean
class AddMonoid' (M : Type) extends AddSemigroup M, AddZeroClass M where
  /-- Multiplication by a natural number. -/
  nsmul : ℕ → M → M := nsmul
  /-- Multiplication by `(0 : ℕ)` gives `0`. -/
  nsmul_zero : ∀ x, nsmul 0 x = 0 := by intros; rfl
  /-- Multiplication by `(n + 1 : ℕ)` behaves as expected. -/
  nsmul_succ : ∀ (n : ℕ) (x), nsmul (n + 1) x = x + nsmul n x := by intros; rfl

instance mySMul {M : Type} [AddMonoid' M] : SMul ℕ M := ⟨AddMonoid'.nsmul⟩
```

```lean
instance : AddMonoid' ℤ where
  add := (· + ·)
  add_assoc := Int.add_assoc
  zero := 0
  zero_add := Int.zero_add
  add_zero := Int.add_zero
  nsmul := fun n m ↦ (n : ℤ) * m
  nsmul_zero := Int.zero_mul
  nsmul_succ := fun n m ↦ show (n + 1 : ℤ) * m = m + n * m
    by rw [Int.add_mul, Int.add_comm, Int.one_mul]
```

---

You are now ready to read the definition of monoids, groups, rings and modules
in Mathlib. There are more complicated than what we have seen here, because they are part of a huge
hierarchy, but all principles have been explained above.

---

## Morphisms

---

## Subobjects

