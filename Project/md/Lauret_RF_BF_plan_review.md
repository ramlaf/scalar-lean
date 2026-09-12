This is now a strong plan. I would use it, but I would make several surgical changes before giving it to Codex. The main remaining issue is that the plan says “finite-dimensional heart first” while some of its basic definitions already depend on the hardest global Lie-theoretic infrastructure.

I checked the plan against Lauret's paper. The scope is accurate: \(\mathcal H_{q,n}\) has exactly conditions (h1)–(h4), Proposition 2.2 has precisely the block-diagonal change of basis with the condition
\[
[h_n^th_n,\operatorname{ad}_\mu\mathfrak k|_{\mathfrak p}]=0,
\]
and Theorem 3.3 gives the diffeomorphisms, both \(h\)-ODEs, (iii), and (iv). The paper also explicitly states in Remark 3.4 that the maximal intervals coincide.

My changes would be:

1. **Split `HomogeneousBracket` into algebraic and global parts.** This is the most important change. Currently you propose
   ```lean
   structure HomogeneousBracket ... where
     ...
     isotropy_closed : IsClosedSubgroup ...
   ```
   but then Phase 3 is supposed to prove the ODE core *before* Phase 4 develops Lie integration. Those two choices are incompatible.

   I would instead have something like
   \[
   \texttt{IsAlgebraicHomogeneousBracket}(\mu)
   \]
   containing (h1), (h3), (h4), and perhaps the reductive parts of (h1), and separately
   \[
   \texttt{HasClosedIsotropy}(\mu)
   \]
   encoding (h2). Then
   \[
   \texttt{IsHomogeneousBracket}(\mu)
   :=
   \texttt{IsAlgebraicHomogeneousBracket}(\mu)
   \land \texttt{HasClosedIsotropy}(\mu).
   \]
   This exactly reflects Lauret's Lemma 3.2: almost all of the proof is algebraic; closedness is transferred only at the very end using the integrated group isomorphism.

2. **Do not make the simply connected integration “canonical”.** The sentence saying the construction should be canonical up to equivalence, or prove independence of the choice, risks creating an unnecessary subproject. Lean's `Classical.choose` is perfectly legitimate here. What you need mathematically is existence and functoriality:
   \[
   (\mathfrak g,\mu)\rightsquigarrow G_\mu,\qquad
   F:\mathfrak g_\mu\simeq\mathfrak g_\lambda
   \rightsquigarrow \widetilde F:G_\mu\simeq G_\lambda.
   \]
   Once a simply connected integration is chosen consistently enough for this theorem, uniqueness of simply connected integration supplies all invariance that matters. There is no need to prove a categorical universal construction merely to formalize Theorem 3.3.

3. **Keep Shi/Chen–Zhu completely outside the critical path.** The current plan mostly recognizes this, but the dependency graph still displays Ricci-flow existence/uniqueness as though it were a dependency of Theorem 3.3. For the theorem you actually want to Lean-verify, I would formulate the main result conditional on supplied solutions of the two finite-dimensional ODEs. The proof printed in Theorem 3.3 is then genuinely finite-dimensional except for interpreting the result geometrically.

   This does not falsify Lauret's theorem. The paper starts with the Ricci solution \(g(t)\), the invariant-inner-product solution, and the bracket-flow solution, and proves their equivalence. The separate assertion that the first exists uniquely by general Ricci-flow theory is surrounding infrastructure, not the novel RF–BF argument. Lauret's introduction indeed invokes preservation of isometries to identify the RF with an ODE on invariant inner products.

   I would therefore make **“formalize Shi/Chen–Zhu” explicitly out of scope** for this project.

4. **Change the representation of varying inner products.** `InnerProductSpaceData p` is likely to fight Lean's typeclass system. An inner product varying with \(t\) should be ordinary data, for example a symmetric bilinear form
   \[
   q_t\in\operatorname{Sym}^2(\mathfrak p^*)
   \]
   together with positive definiteness. Keep the fixed background inner product as the actual `InnerProductSpace` instance.

   Thus use something conceptually like
   ```lean
   structure PosDefForm (p : Type*) where
     form : p →ₗ[ℝ] p →ₗ[ℝ] ℝ
     symmetric : ...
     posDef : ...
   ```
   Then
   \[
   q_t(X,Y)=\langle h(t)X,h(t)Y\rangle
   \]
   is an equality of ordinary bundled objects rather than an equality between changing typeclass instances. This will save a great deal of coercion trouble.

5. **Make the algebraic Ricci covariance theorem primary, rather than relying first on general Riemannian naturality.** For the ODE proof, what you actually need is
   \[
   \Ric_{\tilde h\cdot\mu}
     =h\,\Ric_{q}\,h^{-1}.
   \]
   Since you are already implementing formula (15), this covariance should be provable directly by finite-dimensional algebra. That lets almost the entire proof of Theorem 3.3 compile before you possess a sophisticated general theorem saying Ricci curvature is natural under Riemannian isometries.

   Later, `geometricRicci_eq_algebraicRicci` identifies this algebraic object with genuine geometric Ricci. This gives a cleaner dependency direction:
   \[
   \boxed{\text{algebra first}}
   \longrightarrow
   \boxed{\text{RF--BF theorem}}
   \longrightarrow
   \boxed{\text{geometric interpretation}}.
   \]

6. **I would demote the full Besse 7.38 calculation from the initial critical path.** Ultimately, if “no external unverified theorem” is literal, you do need to prove that formula (15) is the Ricci operator of \(g_\mu\). That could be substantial. But it is logically separable from the RF–BF transport argument.

   There is a valuable milestone at which Lean proves, with `algebraicRicci` defined by formula (15),
   \[
   q(t)=h(t)^*q_0,\qquad
   \mu(t)=\widetilde h(t)\cdot\mu_0,
   \]
   and proves both \(h\)-ODEs. That verifies the entire ODE argument in Theorem 3.3. Only afterwards should you prove that `algebraicRicci` equals Riemannian Ricci.

   I would make this a named milestone rather than allowing the Ricci-formula calculation to block everything downstream.

7. **Your separation of Theorem 3.3 from maximal-interval equality is exactly right.** The paper puts the equality of maximal intervals in Remark 3.4 as a consequence, not literally as one of clauses (i)–(iv). The proposed
   ```lean
   theorem maximalIntervals_eq ...
   ```
   should remain separate. Its proof is particularly clean: either ODE reconstructs the other solution, so extension of either one extends the other.

There is one further subtlety I would emphasize. Your `bracketFlow_stays_in_orbit` theorem is more useful than a generic “vector field tangent to an orbit implies invariance” theorem. Lauret writes “by a standard ODE theory argument” at precisely this point, but in Lean I would **construct the companion \(h(t)\)** and show
\[
\lambda(t)=
\begin{pmatrix}I&0\\0&h(t)\end{pmatrix}\cdot\mu_0
\]
satisfies the bracket ODE. Uniqueness gives \(\lambda=\mu\). This simultaneously gives orbit preservation, an explicit isomorphism, and the \(h(t)\) later required by Theorem 3.3. It avoids formalizing tangent spaces of nonlinear \(\mathrm{GL}(p)\)-orbits merely for Lemma 3.2.

So I would slightly reorganize your implementation order to:

\[
\boxed{\text{bracket/GL linear algebra}}
\to
\boxed{\text{formula (15) algebra}}
\to
\boxed{\text{ODE transport theorem}}
\to
\boxed{\text{algebraic part of Lemma 3.2}}
\]
\[
\to
\boxed{\text{Lie III + quotients}}
\to
\boxed{\text{(h2) + Proposition 2.2}}
\to
\boxed{\text{geometric Ricci identification}}
\to
\boxed{\text{literal Theorem 3.3}}
\to
\boxed{\text{maximal intervals}}.
\]

That ordering has a major practical advantage: **the first genuinely dangerous dependency is pushed very late**. If Mathlib turns out not to have enough Lie III/quotient infrastructure, you still end up with a substantial axiom-free Lean theorem formalizing the entire RF–BF ODE mechanism rather than a half-built global geometry library.

With these changes, I would rate this plan around **9/10 as an executable research formalization plan**. The previous plan's scope risk is largely gone. The two genuine risks remaining are very sharply localized: **Lie III/quotient geometry** and **the identification of formula (15) with geometric Ricci**. Everything else in Lauret's RF–BF argument looks like a realistic finite-dimensional Lean project.
