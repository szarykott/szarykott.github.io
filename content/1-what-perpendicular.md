+++
title = "What is perpendicular programming?"
date = 2021-01-06
+++


Most programmers seem to be obsessed with parallel programming nowadays. Only some know about new paradigm that is emerging, slowly, but inevitably. It is called perpendicular programming. What is this new paradigm and why should you start using is today?

<!-- more -->

### Comparison with parallel programming

Parallel programming is all about executing some instructions in parallel on separate CPU cores as depicted below:


![parallel lines](./../../images/1/parallel.png)


There is a notable problem with this approach. As clearly visible in the picture - threds of execution never meet, making parallel programming paradigm less usable. Perpendicual programming addresses this problem.

![perpendicular lines](./../../images/1/perpendicular.png)

In this new paradigm threads never fail to meet at a certain point in time. What is more then provide values to each other exactly at the time other threads need it, not earlier, not later. This allows for excellent performance.

### Where to read more

Currently this paradigm is not very well known, but it has a bright future. Look for it in newest Microsoft and Google blog posts!


