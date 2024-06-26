---
layout: post
title: Derivation of neutron incoherent scattering length
subtitle:
tags: [scattering,solid state physics,crystallography,total scattering]
author: Yuanpeng Zhang
comments: true
use_math: true
---

<p align='center'>
<img src="/assets/img/posts/nist_neutron_data.png"
   style="border:none;"
   alt="nist_table"
   title="nist_table" />
<br />
<b>Figure. 1.</b> Neutron scattering length table by NIST [1].
</p>

<p style='text-align: justify'>
When we do calculations relevant to neutron scattering, it is unavoidable that we will come across either coherent, incoherent or total scattering length of elements, which describes the strength of scattering neutrons by different elements, either coherently, incoherently or both. In this article, I will try to uncover the three different types of neutron scattering length, by going through the derivation in details.

First, the differential cross section for neutron scattering by a certain structure is given as below -- here we do the derivation for a single-species system so we can derive the relevant scattering length conveniently.
</p>

$$
\frac{d\sigma}{d\Omega} = \sum_{i,j}b_ib_je^{-i\vec{Q}\cdot (\vec{R}_i - \vec{R}_j)}
$$

<p style='text-align: justify'>
where $$i$$, $$j$$ refers to atoms located at different locations in the structure under beam. In practice, every single $$b_i$$, $$b_j$$ depends on the corresponding nuclear isotope, spin orientation relative to neutron, nuclear eigenstate, etc. What we are interested in and what is meaningful in practical calculation is the average $$\langle b_ib_j\rangle$$ across all pairs in system, which then can be used as an effective representative of system. To obtain $$\langle b_ib_j\rangle$$, we first write,
</p>

$$
b_i = \langle b\rangle + \delta b_i
$$

<p style='text-align: justify'>
where we should have $$\langle\delta b_i\rangle = 0$$, if assuming random variation of $$b_i$$ around $$\langle b\rangle$$ for an arbitrary $$i$$. Further,
</p>

$$
\begin{equation}\begin{aligned}\langle b_ib_j\rangle & = \Big\langle\langle b\rangle^2 + \langle b\rangle(\delta b_i + \delta b_j) + \delta b_i\delta b_j\Big\rangle\\ & = \Big\langle\langle b\rangle^2\Big\rangle + \big\langle\langle b\rangle(\delta b_i + \delta b_j)\big\rangle + \langle\delta b_i\delta b_j\rangle\\ & = \langle b\rangle^2 + 0 + \langle\delta b_i \delta b_j\rangle\end{aligned}\nonumber\end{equation}
$$

<p style='text-align: justify'>
$$\langle\delta b_i\delta b_j\rangle$$ is then the quantity we want to calculate. The good news is, it is only non-zero when $$i = j$$, assuming atom $$i$$ and $$j$$ is irrelevant to each other. For $$i = j$$, we have,
</p>

$$
\langle b^2\rangle = \langle b\rangle^2 + \langle\delta b_i^2\rangle \Rightarrow \langle\delta b_i^2\rangle = \langle b^2\rangle - \langle b\rangle^2
$$

<p style='text-align: justify'>
For $$i \neq j$$, we simply have,
</p>

$$
\langle b_ib_j\rangle = \langle b\rangle^2
$$

<p style='text-align: justify'>
Going back to the equation for calculating differential cross section and putting in the expression for $$\langle b_ib_j\rangle$$, we have,
</p>

$$
\begin{equation}\begin{aligned}\frac{d\sigma}{d\Omega} & = \sum_{i,j}^N\langle b_ib_j\rangle e^{i\vec{Q}\cdot(\vec{R}_i - \vec{R}_j)}\\ & = \sum_{i\neq j}^N\Big[\langle b\rangle^2e^{i\vec{Q}\cdot(\vec{R}_i - \vec{R}_j)} + \langle\delta b_i\delta b_j\rangle e^{i\vec{Q}\cdot(\vec{R}_i - \vec{R}_j)}\Big] + \\ & \hspace{0.75cm} \sum_{i=j}^N\Big[\langle b\rangle^2e^{i\vec{Q}\cdot(\vec{R}_i - \vec{R}_j)} + \langle\delta b_i\delta b_j\rangle e^{i\vec{Q}\cdot(\vec{R}_i - \vec{R}_j)}\Big]\\ & = \sum_{i\neq j}^N\Big[\langle b\rangle^2e^{i\vec{Q}\cdot(\vec{R}_i - \vec{R}_j)} + 0\Big] + \sum_{i=j}^N\Big[\langle b\rangle^2e^{i\vec{Q}\cdot(\vec{R}_i - \vec{R}_j)} + (\langle b^2\rangle - \langle b\rangle^2)\Big]\\ & = \sum_{i,j}^N\langle b\rangle^2e^{i\vec{Q}\cdot(\vec{R}_i - \vec{R}_j)} + N(\langle b^2\rangle - \langle b\rangle^2)\end{aligned}\nonumber\end{equation}
$$

<p style='text-align: justify'>
where $$N$$ is the total number of atoms in system. Here we see the first term is relevant to atomic positions and therefore is the coherent scattering term. Accordingly, $$\langle b\rangle^2$$ is the coherent scattering length (squared). The second term is nothing to do with structure and therefore is the incoherent scattering term. Accordingly,
</p>

$$
\langle b^2\rangle - \langle b\rangle^2
$$

<p style='text-align: justify'>
is the incoherent scattering length (squared).
</p>

<br />

<b>References</b>

[1] [https://www.ncnr.nist.gov/resources/n-lengths/](https://www.ncnr.nist.gov/resources/n-lengths/)
