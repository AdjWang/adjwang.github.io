---
title: "Quaternion"
summary: "Use glm::quat as example"
date: 2024-06-09T21:40:03+08:00
math: true
table_of_contents: true
---

## Hamiltons Quaternions

$$
\begin{equation} \label{eq:quat_basic}
q = w + x\vec{i} + y\vec{j} + z\vec{k}
\end{equation}
$$

式(\ref{eq:quat_basic})中 $\vec{i}$, $\vec{j}$, $\vec{k}$ 视为与 x, y, z 轴重合的单位向量，计算时要考虑方向。

$$
\begin{align}
\vec{i} \cdot \vec{j} = \vec{i} \cdot \vec{k} = \vec{j} \cdot \vec{k} = 0 \label{eq:ijk_first} \\
\vec{i} \times \vec{i} = \vec{j} \times \vec{j} = \vec{k} \times \vec{k} = -1 \\
\vec{i} \times \vec{j} = -\vec{j} \times \vec{i} = \vec{k} \\
\vec{j} \times \vec{k} = -\vec{k} \times \vec{j} = \vec{i} \\
\vec{k} \times \vec{i} = -\vec{i} \times \vec{k} = \vec{j} \label{eq:ijk_last}
\end{align}
$$

Quaternion 是虚数，包含 1 个实部和 3 个虚部，在计算机中存储四个系数 $w$, $x$, $y$, $z$，类型为浮点数。由于历史原因，quaternion 在 glm 中的数据存储顺序有歧义，可以通过宏定义 GLM_FORCE_QUAT_DATA_WXYZ 指定数据存储顺序，源码实现如下：

```c++
// glm\detail\type_quat.inl
template <typename T, qualifier Q>
GLM_FUNC_QUALIFIER GLM_CONSTEXPR qua<T, Q>::qua(T _w, T _x, T _y, T _z)
#ifdef GLM_FORCE_QUAT_DATA_WXYZ
	: w(_w), x(_x), y(_y), z(_z)
#else
	: x(_x), y(_y), z(_z), w(_w)
#endif
{}
```

## 从欧拉角构造

{{< image_with_caption src="./quat_rotate.svg" width=25% >}}

$$
\begin{equation} \label{eq:quat_from_euler}
q = s + \vec{u} = \cos \frac{\theta}{2} + (x\vec{i} + y\vec{j} + z\vec{k})\sin \frac{\theta}{2}
\end{equation}
$$

式(\ref{eq:quat_from_euler})表示绕空间内某个向量$\vec{v}=(x, y, z)$旋转角度$\theta$所对应的 quaternion.

可以通过累乘 quaternion 来表示绕 3 个轴的旋转操作，注意 quaternion 乘法不满足交换律，先后绕轴的顺序会影响最终结果，因此同一个三维旋转对应的欧拉角表示不唯一，在不同的数学库中采用不同的规范。在 unity 中，旋转顺序为 ZXY {{< cite name="unity_quat" >}}；在 glm 中，旋转顺序为 ZYX {{< cite name="glm_quat_constructor" >}}。

{{< highlight_with_caption lang="c++" caption="glm::quat 输入欧拉角的构造函数" label="lst:quatconstruct">}}
// glm\detail\type_quat.inl
template<typename T, qualifier Q>
GLM_FUNC_QUALIFIER GLM_CONSTEXPR qua<T, Q>::qua(vec<3, T, Q> const& eulerAngle)
{
    vec<3, T, Q> c = glm::cos(eulerAngle * T(0.5));
    vec<3, T, Q> s = glm::sin(eulerAngle * T(0.5));

    this->w = c.x * c.y * c.z + s.x * s.y * s.z;
    this->x = s.x * c.y * c.z - c.x * s.y * s.z;
    this->y = c.x * s.y * c.z + s.x * c.y * s.z;
    this->z = c.x * c.y * s.z - s.x * s.y * c.z;
}
{{</ highlight_with_caption >}}

绕 ZYX 轴旋转的推导过程如下{{< cite name="quat_conversion_wiki" >}}{{< cite name="quat_conversion_euler" >}}：

假设 $\alpha$, $\beta$, $\gamma$ 分别代表物体绕 x, y, z 轴上单位向量的旋转角度，那么对应的 3 个 quaternion 就是：

$$
\begin{align}
q_\alpha &= \cos \frac{\alpha}{2} + \vec{i}\sin \frac{\alpha}{2} \\
q_\beta &= \cos \frac{\beta}{2} + \vec{j}\sin \frac{\beta}{2} \\
q_\gamma &= \cos \frac{\gamma}{2} + \vec{k}\sin \frac{\gamma}{2}
\end{align}
$$

按照 ZYX 旋转顺序累乘得到最终的 quaternion，其叉乘公式如下：

$$
\begin{align} \label{eq:quat_mul}
q_1 \times q_2 = (s_1s_2 - \vec{u}_1 \cdot \vec{u}_2) + (s_1\vec{u}_2 + s_2\vec{u}_1 + \vec{u}_1 \times \vec{u}_2)
\end{align}
$$

由于坐标轴是互相垂直的，结合式(\ref{eq:ijk_first})，上式中的点乘部分等于零，所以此处 quaternion 叉乘简化为：

$$
\begin{align}
q_1 \times q_2 = s_1s_2 + s_1\vec{u}_2 + s_2\vec{u}_1 + \vec{u}_1 \times \vec{u}_2
\end{align}
$$

据此计算整体结果：

$$
\begin{flalign}
\nonumber
q &= q_\gamma \times q_\beta \times q_\alpha \\
\nonumber
&= (\cos \frac{\gamma}{2} + \vec{k}\sin \frac{\gamma}{2}) \times (\cos \frac{\beta}{2} + \vec{j}\sin \frac{\beta}{2}) \times (\cos \frac{\alpha}{2} + \vec{i}\sin \frac{\alpha}{2}) \\
\nonumber
&= (\cos \frac{\gamma}{2} \cdot \cos \frac{\beta}{2} + \cos \frac{\gamma}{2} \cdot \vec{j}\sin \frac{\beta}{2} + \cos \frac{\beta}{2} \cdot \vec{k}\sin \frac{\gamma}{2} + (\vec{k} \times \vec{j}) \sin \frac{\gamma}{2} \sin \frac{\beta}{2}) \\
\nonumber
&\quad \times (\cos \frac{\alpha}{2} + \vec{i}\sin \frac{\alpha}{2}) \\
\nonumber
&= (\cos \frac{\gamma}{2} \cos \frac{\beta}{2} + \vec{j} \cos \frac{\gamma}{2} \sin \frac{\beta}{2} + \vec{k} \cos \frac{\beta}{2} \sin \frac{\gamma}{2} - \vec{i} \sin \frac{\gamma}{2} \sin \frac{\beta}{2}) \\
\nonumber
&\quad \times (\cos \frac{\alpha}{2} + \vec{i} \sin \frac{\alpha}{2}) \\
\nonumber
&= \cos \frac{\gamma}{2} \cos \frac{\beta}{2} \cos \frac{\alpha}{2} + \vec{j} \cos \frac{\gamma}{2} \sin \frac{\beta}{2} \cos \frac{\alpha}{2} + \vec{k} \cos \frac{\beta}{2} \sin \frac{\gamma}{2} \cos \frac{\alpha}{2} - \vec{i} \sin \frac{\gamma}{2} \sin \frac{\beta}{2} \cos \frac{\alpha}{2} \\
\nonumber
&\quad + \vec{i} \cos \frac{\gamma}{2} \cos \frac{\beta}{2} \sin \frac{\alpha}{2} + (\vec{j} \times \vec{i}) \cos \frac{\gamma}{2} \sin \frac{\beta}{2} \sin \frac{\alpha}{2} + (\vec{k} \times \vec{i}) \cos \frac{\beta}{2} \sin \frac{\gamma}{2} \sin \frac{\alpha}{2} - (\vec{i} \times \vec{i}) \sin \frac{\gamma}{2} \sin \frac{\beta}{2} \sin \frac{\alpha}{2} \\
\nonumber
&= \cos \frac{\alpha}{2} \cos \frac{\beta}{2} \cos \frac{\gamma}{2} + \sin \frac{\alpha}{2} \sin \frac{\beta}{2} \sin \frac{\gamma}{2} \\
\nonumber
&\quad + \vec{i} ( \sin \frac{\alpha}{2} \cos \frac{\beta}{2} \cos \frac{\gamma}{2} - \cos \frac{\alpha}{2} \sin \frac{\beta}{2} \sin \frac{\gamma}{2} ) \\
\nonumber
&\quad + \vec{j} ( \cos \frac{\alpha}{2} \sin \frac{\beta}{2} \cos \frac{\gamma}{2} + \sin \frac{\alpha}{2} \cos \frac{\beta}{2} \sin \frac{\gamma}{2} ) \\
\nonumber
&\quad + \vec{k} ( \cos \frac{\alpha}{2} \cos \frac{\beta}{2} \sin \frac{\gamma}{2} - \sin \frac{\alpha}{2} \sin \frac{\beta}{2} \cos \frac{\gamma}{2} )
\end{flalign}
$$

上式结果与{{< reference label="lst:quatconstruct" >}}相同。类似的，如果按照 ZXY 顺序旋转得到：

$$
\begin{flalign}
\nonumber
q &= q_\gamma \times q_\alpha \times q_\beta \\
\nonumber
&= \cos \frac{\alpha}{2} \cos \frac{\beta}{2} \cos \frac{\gamma}{2} - \sin \frac{\alpha}{2} \sin \frac{\beta}{2} \sin \frac{\gamma}{2} \\
\nonumber
&\quad + \vec{i} ( \sin \frac{\alpha}{2} \cos \frac{\beta}{2} \cos \frac{\gamma}{2} - \cos \frac{\alpha}{2} \sin \frac{\beta}{2} \sin \frac{\gamma}{2} ) \\
\nonumber
&\quad + \vec{j} ( \cos \frac{\alpha}{2} \sin \frac{\beta}{2} \cos \frac{\gamma}{2} + \sin \frac{\alpha}{2} \cos \frac{\beta}{2} \sin \frac{\gamma}{2} ) \\
\nonumber
&\quad + \vec{k} ( \cos \frac{\alpha}{2} \cos \frac{\beta}{2} \sin \frac{\gamma}{2} + \sin \frac{\alpha}{2} \sin \frac{\beta}{2} \cos \frac{\gamma}{2} )
\end{flalign}
$$

## 乘法

{{< highlight_with_caption lang="c++" caption="glm::quat 乘法" label="lst:quatmul">}}
// glm\detail\type_quat.inl
template<typename T, qualifier Q>
template<typename U>
GLM_FUNC_QUALIFIER GLM_CONSTEXPR qua<T, Q> & qua<T, Q>::operator*=(qua<U, Q> const& r)
{
    qua<T, Q> const p(*this);
    qua<T, Q> const q(r);

    this->w = p.w * q.w - p.x * q.x - p.y * q.y - p.z * q.z;
    this->x = p.w * q.x + p.x * q.w + p.y * q.z - p.z * q.y;
    this->y = p.w * q.y + p.y * q.w + p.z * q.x - p.x * q.z;
    this->z = p.w * q.z + p.z * q.w + p.x * q.y - p.y * q.x;
    return *this;
}
{{</ highlight_with_caption >}}

把式(\ref{eq:quat_basic})代入式(\ref{eq:quat_mul})得到叉乘结果：

$$
\begin{flalign}
\nonumber
q_1 \times q_2 &= (s_1s_2 - \vec{u}_1 \cdot \vec{u}_2) + (s_1\vec{u}_2 + s_2\vec{u}_1 + \vec{u}_1 \times \vec{u}_2) \\
\nonumber
&= w_1w_2 - (x_1x_2 + y_1y_2 + z_1z_2) \\
\nonumber
&\quad + w_1x_2\vec{i} + w_1y_2\vec{j} + w_1z_2\vec{k} \\
\nonumber
&\quad + w_2x_1\vec{i} + w_2y_1\vec{j} + w_2z_1\vec{k} \\
\nonumber
&\quad + \begin{vmatrix}
\vec{i} & \vec{j} & \vec{k} \\ 
x_1 & y_1 & z_1 \\ 
x_2 & y_2 & z_2
\end{vmatrix} \\
\nonumber
&= w_1w_2 - x_1x_2 - y_1y_2 - z_1z_2 \\
\nonumber
&\quad + (w_1x_2 + x_1w_2 + y_1z_2 - z_1y_2)\vec{i} \\
\nonumber
&\quad + (w_1y_2 + y_1w_2 + z_1x_2 - x_1z_2)\vec{j} \\
\nonumber
&\quad + (w_1z_2 + z_1w_2 + x_1y_2 - y_1x_2)\vec{k}
\end{flalign}
$$

## 旋转向量

{{< highlight_with_caption lang="c++" caption="glm::quat 旋转向量" label="lst:quatrotate">}}
// glm\detail\type_quat.inl
template<typename T, qualifier Q>
GLM_FUNC_QUALIFIER GLM_CONSTEXPR vec<3, T, Q> operator*(qua<T, Q> const& q, vec<3, T, Q> const& v)
{
    vec<3, T, Q> const QuatVector(q.x, q.y, q.z);
    vec<3, T, Q> const uv(glm::cross(QuatVector, v));
    vec<3, T, Q> const uuv(glm::cross(QuatVector, uv));

    return v + ((uv * q.w) + uuv) * static_cast<T>(2);
}

template<typename T, qualifier Q>
GLM_FUNC_QUALIFIER GLM_CONSTEXPR vec<3, T, Q> operator*(vec<3, T, Q> const& v, qua<T, Q> const& q)
{
    return glm::inverse(q) * v;
}
{{</ highlight_with_caption >}}

$$
\begin{flalign}
qvq^* &= (s + \vec{u}) \times (0 + \vec{v}) \times (s - \vec{u}) \label{eq:qvqstar_quatvec} \\
\nonumber
&= (- \vec{u} \cdot \vec{v} + s\vec{v} + \vec{u} \times \vec{v}) \times (s - \vec{u}) \\
\nonumber
&= - s\vec{u} \cdot \vec{v} + (s\vec{v} + \vec{u} \times \vec{v}) \cdot \vec{u} \\
&\quad + (\vec{u} \cdot \vec{v})\vec{u} + s(s\vec{v} + \vec{u} \times \vec{v}) - (s\vec{v} + \vec{u} \times \vec{v}) \times \vec{u} \label{eq:qvqstar_simplify} \\
&= 2s\vec{u} \times \vec{v} + \vec{u} \times (\vec{u} \times \vec{v}) + (\vec{u} \cdot \vec{v}) \vec{u} + s^2\vec{v} \label{eq:qvqstar_half}
\end{flalign}
$$

式(\ref{eq:qvqstar_simplify})化简时会遇到 $(\vec{u} \times \vec{v}) \cdot \vec{u}$，该项等同于 $\vec{v} \cdot (\vec{u} \times \vec{u})$，等于零{{< cite name="evans2000vecmath" >}}。

单位 quaternion 满足 $s^2 + \vec{u}\cdot\vec{u} = 1$，即 $s^2 = 1 - \vec{u}\cdot\vec{u}$. 另外，根据向量叉乘规则{{< cite name="evans2000vecmath" >}} $\vec{a} \times (\vec{b} \times \vec{c}) = (\vec{a} \cdot \vec{c})\vec{b} - (\vec{a} \cdot \vec{b})\vec{c}$ 可得 $\vec{u} \times (\vec{u} \times \vec{v}) = (\vec{u} \cdot \vec{v})\vec{u} - (\vec{u} \cdot \vec{u})\vec{v}$，代入式(\ref{eq:qvqstar_half})，继续推导：

$$
\begin{flalign}
\nonumber
qvq^* &= 2s\vec{u} \times \vec{v} + \vec{u} \times (\vec{u} \times \vec{v}) + (\vec{u} \cdot \vec{v}) \vec{u} + (1 - \vec{u}\cdot\vec{u})\vec{v} \\
\nonumber
&= 2s\vec{u} \times \vec{v} + \vec{u} \times (\vec{u} \times \vec{v}) + ((\vec{u} \cdot \vec{v}) \vec{u} - (\vec{u}\cdot\vec{u})\vec{v}) + \vec{v} \\
&= \vec{v} + 2s\vec{u} \times \vec{v} + 2\vec{u} \times (\vec{u} \times \vec{v}) \label{eq:qvqstar_crossres}
\end{flalign}
$$

上式结果与{{< reference label="lst:quatrotate" >}}相同。在一些资料{{< cite name="jbk1999qrs" >}}里使用另一种化简形式，可通过续接式(\ref{eq:qvqstar_half})展开 $\vec{u} \times \vec{u} \times \vec{v}$ 得到：

$$
\begin{flalign}
\nonumber
qvq^* &= 2s\vec{u} \times \vec{v} + \vec{u} \times (\vec{u} \times \vec{v}) + (\vec{u} \cdot \vec{v}) \vec{u} + s^2\vec{v} \\
\nonumber
&= 2s\vec{u} \times \vec{v} + (\vec{u} \cdot \vec{v})\vec{u} - (\vec{u} \cdot \vec{u})\vec{v} + (\vec{u} \cdot \vec{v}) \vec{u} + s^2\vec{v} \\
&= (s^2 - \vec{u} \cdot \vec{u})\vec{v} + 2(\vec{u}\cdot\vec{v})\vec{u} + 2s(\vec{u}\times\vec{v}) \label{eq:qvqstar_dotres}
\end{flalign}
$$

## 旋转矩阵

旋转矩阵用于在实数空间里旋转一个向量，程序里的常见做法是将 quaternion 转换为旋转矩阵然后放到模型变换矩阵中，交给显卡计算模型顶点位置。

{{< highlight_with_caption lang="c++" caption="glm::quat 转换旋转矩阵" label="lst:quatmatcast">}}
// glm\gtc\quaternion.inl
template<typename T, qualifier Q>
GLM_FUNC_QUALIFIER mat<3, 3, T, Q> mat3_cast(qua<T, Q> const& q)
{
    mat<3, 3, T, Q> Result(T(1));
    T qxx(q.x * q.x);
    T qyy(q.y * q.y);
    T qzz(q.z * q.z);
    T qxz(q.x * q.z);
    T qxy(q.x * q.y);
    T qyz(q.y * q.z);
    T qwx(q.w * q.x);
    T qwy(q.w * q.y);
    T qwz(q.w * q.z);

    Result[0][0] = T(1) - T(2) * (qyy +  qzz);
    Result[0][1] = T(2) * (qxy + qwz);
    Result[0][2] = T(2) * (qxz - qwy);

    Result[1][0] = T(2) * (qxy - qwz);
    Result[1][1] = T(1) - T(2) * (qxx +  qzz);
    Result[1][2] = T(2) * (qyz + qwx);

    Result[2][0] = T(2) * (qxz + qwy);
    Result[2][1] = T(2) * (qyz - qwx);
    Result[2][2] = T(1) - T(2) * (qxx +  qyy);
    return Result;
}

template<typename T, qualifier Q>
GLM_FUNC_QUALIFIER mat<4, 4, T, Q> mat4_cast(qua<T, Q> const& q)
{
    return mat<4, 4, T, Q>(mat3_cast(q));
}
{{</ highlight_with_caption >}}

旋转矩阵可从旋转向量小节旋转向量操作的公式中推导，点乘计算比叉乘简单，所以下面从式(\ref{eq:qvqstar_crossres})和式(\ref{eq:qvqstar_dotres})中选择式(\ref{eq:qvqstar_dotres})推导旋转矩阵。

首先赋予被旋转向量 $\vec{v}$ 在虚数空间中的位置参数 $v_x, v_y, v_z$，使其具有和式(\ref{eq:quat_basic})中的 quaternion 一样的形式，这里也是对式(\ref{eq:qvqstar_quatvec})的进一步细化：

$$
\begin{equation} \label{eq:quat_vec}
\vec{v} = 0 + v_x\vec{i} + v_y\vec{j} + v_z\vec{k}
\end{equation}
$$

然后使用式(\ref{eq:quat_basic})的符号展开简写的实数 $s$ 和向量 $\vec{u}$：

$$
\begin{equation} \label{eq:quat_extbasic}
q = s + \vec{u} = w + x\vec{i} + y\vec{j} + z\vec{k}
\end{equation}
$$

将式(\ref{eq:quat_vec})和式(\ref{eq:quat_extbasic})代入(\ref{eq:qvqstar_dotres})继续推导：

$$
\begin{flalign}
\nonumber
qvq^* &= (s^2 - \vec{u} \cdot \vec{u})\vec{v} + 2(\vec{u}\cdot\vec{v})\vec{u} + 2s(\vec{u}\times\vec{v}) \\
\nonumber
&= (w^2-x^2-y^2-z^2)(v_x\vec{i} + v_y\vec{j} + v_z\vec{k}) \\
\nonumber
&\quad + 2(xv_x + yv_y +zv_z)(x\vec{i} + y\vec{j} + z\vec{k}) \\
\nonumber
&\quad + 2w\left[(yv_z-zv_y)\vec{i} + (zv_x-xv_z)\vec{j} + (xv_y-yv_x)\vec{k}\right] \\
\nonumber
&= \left[(w^2+x^2-y^2-z^2)v_x + 2(xy-wz)v_y + 2(xz+wy)v_z\right]\vec{i} \\
\nonumber
&\quad + \left[2(xy+wz)v_x + (w^2-x^2+y^2-z^2)v_y + 2(yz-wx)v_z\right]\vec{j} \\
\nonumber
&\quad + \left[2(xz-wy)v_x + 2(yz+wx)v_y + (w^2-x^2-y^2+z^2)v_z\right]\vec{k} \\
\nonumber
&= \begin{pmatrix}
\vec{i} & \vec{j} & \vec{k}
\end{pmatrix} \begin{pmatrix}
w^2+x^2-y^2-z^2 & 2(xy-wz)        & 2(xz+wy) \\
2(xy+wz)        & w^2-x^2+y^2-z^2 & 2(yz-wx) \\
2(xz-wy)        & 2(yz+wx)        & w^2-x^2-y^2+z^2
\end{pmatrix} \begin{pmatrix}
v_x \\
v_y \\
v_z
\end{pmatrix}
\end{flalign}
$$

因此旋转矩阵即为：

$$
\begin{equation} \label{eq:quat_rotmat_nonunit}
R = \begin{pmatrix}
w^2+x^2-y^2-z^2 & 2(xy-wz)        & 2(xz+wy) \\
2(xy+wz)        & w^2-x^2+y^2-z^2 & 2(yz-wx) \\
2(xz-wy)        & 2(yz+wx)        & w^2-x^2-y^2+z^2
\end{pmatrix}
\end{equation}
$$

单位 quaternion 满足 $w^2+x^2+y^2+z^2 = 1$，此时旋转矩阵也可以写成：

$$
\begin{equation} \label{eq:quat_rotmatunit}
R = \begin{pmatrix}
1-2(y^2+z^2) & 2(xy-wz)     & 2(xz+wy) \\
2(xy+wz)     & 1-2(x^2+z^2) & 2(yz-wx) \\
2(xz-wy)     & 2(yz+wx)     & 1-2(x^2+y^2)
\end{pmatrix}
\end{equation}
$$

式(\ref{eq:quat_rotmatunit})与{{< reference label="lst:quatmatcast" >}}旋转矩阵构造过程相同，注意 glm 每个内存行存储矩阵的列向量。

## 欧拉角变换

{{< highlight_with_caption lang="c++" caption="glm::quat 转换欧拉角" label="lst:quateulercast">}}
// glm\gtc\quaternion.inl
template<typename T, qualifier Q>
GLM_FUNC_QUALIFIER vec<3, T, Q> eulerAngles(qua<T, Q> const& x)
{
    return vec<3, T, Q>(pitch(x), yaw(x), roll(x));
}

template<typename T, qualifier Q>
GLM_FUNC_QUALIFIER T roll(qua<T, Q> const& q)
{
    return static_cast<T>(atan(static_cast<T>(2) * (q.x * q.y + q.w * q.z), q.w * q.w + q.x * q.x - q.y * q.y - q.z * q.z));
}

template<typename T, qualifier Q>
GLM_FUNC_QUALIFIER T pitch(qua<T, Q> const& q)
{
    //return T(atan(T(2) * (q.y * q.z + q.w * q.x), q.w * q.w - q.x * q.x - q.y * q.y + q.z * q.z));
    T const y = static_cast<T>(2) * (q.y * q.z + q.w * q.x);
    T const x = q.w * q.w - q.x * q.x - q.y * q.y + q.z * q.z;

    if(all(equal(vec<2, T, Q>(x, y), vec<2, T, Q>(0), epsilon<T>()))) //avoid atan2(0,0) - handle singularity - Matiis
        return static_cast<T>(static_cast<T>(2) * atan(q.x, q.w));

    return static_cast<T>(atan(y, x));
}

template<typename T, qualifier Q>
GLM_FUNC_QUALIFIER T yaw(qua<T, Q> const& q)
{
    return asin(clamp(static_cast<T>(-2) * (q.x * q.z - q.w * q.y), static_cast<T>(-1), static_cast<T>(1)));
}
{{</ highlight_with_caption >}}

假设 $\alpha, \beta, \gamma$ 分别是俯仰角(pitch)，偏航角(yaw)和滚动角(roll)，轴线分别是 $x, y, z$，使用欧拉角构造旋转矩阵得到{{< cite name="quat_conversion_wiki" >}}：

$$
\begin{equation}
R = \begin{pmatrix}
\cos \gamma \cos \beta & \cos \gamma \sin \beta \sin \alpha - \sin \gamma \cos \alpha & \cos \gamma \sin \beta \cos \alpha + \sin \gamma \sin \alpha \\
\sin \gamma \cos \beta & \sin \gamma \sin \beta \sin \alpha + \cos \gamma \cos \alpha & \sin \gamma \sin \beta \cos \alpha - \cos \gamma \sin \alpha \\
-\sin \beta & \cos \beta \sin \alpha & \cos \beta \cos \alpha
\end{pmatrix}
\end{equation}
$$

对应式(\ref{eq:quat_rotmat_nonunit})，可以得到从 quaternion 到欧拉角的变换公式：

$$
\begin{align}
\alpha = \arctan \frac{2(yz+wx)}{w^2-x^2-y^2+z^2} \\
\beta = \arcsin (-2)(xz-wy) \\
\gamma = \arctan \frac{2(xy+wz)}{w^2+x^2-y^2-z^2}
\end{align}
$$

上式结果与{{< reference label="lst:quateulercast" >}}相同。

在变换欧拉角前要三思，Quaternion 有两个非常变态的特性：

- 使用某一欧拉角集合转换成 quaternion 再转换成欧拉角，输出结果与构造时的输入可能不一致{{< cite name="quat_conversion_euler" >}}。

- 由于 atan 函数值域没有 90° 角，该点无法从 quaternion 转换为欧拉角，俗称“奇点”，在旋转矩阵里也表现为万向节锁(Gimbal Lock)。

综合以上两点考虑，在程序中最好只进不出，尽可能避免将 quaternion 转换为欧拉角，如果需要欧拉角形式，直接缓存角度变量。

## 参考文献

{{< bib_list src="./reference.bib" >}}
