# Parallel optimization

Optimization using gradient method and openMP.

**Optimization problem:**

The city is located in a square with coordinates (-10 < x < 10, -10 < y < 10). The city has n stores (n > 3) whose coordinates are known. It is planned to build another m (m > 3) amount of stores. The suitability factor of the shop is assessed by the distance to other stores and the city limit. You need to find coordinates of the new shops in such way that the sum of the suitability factors would be as low as possible.

The effect of the distance between 2 shops on the suitability factor is calculated using this formula:

```math
C(x_{1},y_{1},x_{2},y_{2})=exp(-0.1*((x_{1}-x_{2})^2+(y_{1}-y_{2})^2))
```
The effect of the distance between the shop and the nearest point of the city on the suitability factor is calculated using this formula:

```math
C_{r}(x_{1},y_{1},x_{r},y_{r})=\left\{\begin{matrix}
0,\,if\,shop\,is\,out\,of\,city\,bounds
\\ 
0.5*((x_{1}-x_{2})^2+(y_{1}-y_{2})^2),\,other\,cases
\end{matrix}\right.
```

![uzd](https://user-images.githubusercontent.com/60033715/212542930-b2e9972e-da76-40c3-ae88-1a10f330ce3a.png)

#

**Algorithm explanation:**

The initial coordinates of the shops to be built are distributed in parallel to the threads in equal numbers. For each shop, a suitability factor is calculated and added to the local variable in the thread. A gradient is calculated for the shop and the shop is moved in the opposite direction by a given step (alfa).
```
double curr_factor = 0;
double new_factor = 0;

#pragma omp for
for (int i = 0; i < starting_pos.size(); i++)
{
	curr_factor += suitability_factor(NULL, starting_pos[i].x, starting_pos[i].y, all_shops);

	tuple<double, double> gradient = gradients(starting_pos[i].x, starting_pos[i].y, shops);
	double nrm = norm(gradient);
	double gradient_norm[] = { get<0>(gradient) / nrm, get<1>(gradient) / nrm };
	moved[i].x = all_shops[i].x - alfa * gradient_norm[0];
	moved[i].y = all_shops[i].y - alfa * gradient_norm[1];
}
```
A barrier is called to make sure that all threads have moved all their shops. A new suitability factor is then calculated.
```
#pragma omp barrier

#pragma omp for
for (int i = 0; i < starting_pos.size(); i++)
{
	new_factor += suitability_factor(NULL, moved[i].x, moved[i].y, moved);
}
```
Each thread enters the critical section and adds the suitability factors to the global variables.
```
#pragma omp critical
{
	factor += curr_factor;
	next_factor += new_factor;
}

#pragma omp barrier
```
One thread will check whether the suitability factor has decreased. If it has decreased, the new coordinates of the shops are saved and the current suitability factor variable becomes the previous factor. If the suitability factor has increased, the step to move is halved.
```
#pragma omp single
{
	#pragma omp critical
	{
		counter++;
		if (next_factor < factor)
		{
			all_shops = moved;
			factor = next_factor;
			next_factor = 0;
		}
		else
		{
			alfa /= 2;
			moved = all_shops;
		}
	}
}

#pragma omp barrier
```
This is repeated until the step reaches the minimum specified value or until the maximum number of iterations is reached.

#

**Data files:**

Data file names indicate how many shops are already built (first number) and how many need to be built (second number).

example: 
50_100.txt

50 already existing shop coordinates and 100 shop coordinates that need to optimized. Both sets of coordinates are seperated by a symbol #.

Result file contains original and mooved coordinates of shops.
