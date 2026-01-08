# Face Detection via Jacobian Optimization 🎭

[![Face Detection Model CI](https://github.com/RohitKattimani/JacobianModel/actions/workflows/main.yml/badge.svg)](https://github.com/RohitKattimani/JacobianModel/actions)

An implementation of a high-precision face detection and landmark localization system that leverages **Jacobian matrices** for geometric alignment and gradient-based optimization.

## Project Overview

Traditional face detection often relies on rigid sliding windows or deep learning heuristics. This project implements a more mathematically grounded approach by treating face detection as a **non-linear least squares problem**. 

By calculating the **Jacobian matrix ($J$)**, we can determine how the error between our model's "mean face" and the target image changes with respect to spatial transformations (scaling, rotation, and translation).

### Why use Jacobians?
* **Geometric Sensitivity:** Understand exactly how small movements in landmark points affect the overall fit.
* **Fast Convergence:** Enables the use of the **Gauss-Newton** or **Levenberg-Marquardt** algorithms for real-time tracking.
* **Sub-pixel Precision:** Achieves higher accuracy in landmark placement than standard bounding-box regressors.

---

## Mathematical Foundation

At the core of our solver is the relationship between the image intensities and the parameter updates. We define our Jacobian as:



$$J = \frac{\partial \mathbf{f}}{\partial \mathbf{p}}$$

Where:
- $\mathbf{f}$ is the vector of facial landmark coordinates.
- $\mathbf{p}$ represents the shape parameters (the "warping" of the face).

The update step is calculated by solving the normal equations:
$$\Delta \mathbf{p} = -(J^T J + \lambda I)^{-1} J^T \mathbf{r}$$

---

## Installation

1. **Clone the repository:**
   ```bash
   git clone [https://github.com/YOUR_USERNAME/YOUR_REPO_NAME.git](https://github.com/YOUR_USERNAME/YOUR_REPO_NAME.git)
   cd YOUR_REPO_NAME
