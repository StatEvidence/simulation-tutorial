<!-- <style> -->
<!-- blockquote { -->
<!-- background-color: #a6bddb50; -->
<!-- font-family: "helvetica"; -->
<!-- font-size:110%; -->
<!-- } -->
<!-- blockquote li{ -->
<!-- padding-bottom:10px; -->
<!-- } -->
<!-- svg { -->
<!-- width:none; -->
<!-- height:none; -->
<!-- } -->
<!-- </style> -->
How to simulate operating characteristics
=========================================

What is an operating characteristic
-----------------------------------

text here

General advice for computing tasks
----------------------------------

The questions in the previous section require you to perform a simulation study. Please remember the tgs axioms of computing:

1.  **The act of turning on the computer does not magically endow you with understanding of your task.** If you do not know how you will perform an analysis or simulation before you turn on your computer, you will not know how to do it afterwards either.

2.  **Use modular/functional programming.** Functional programming means that you identify and write short, single purpose functions for each distinct task in your program. (Examples below.) This will allow you to develop your code in a systematic way, and it will provide a natural method for debugging your code. You will simply need to verify that the different sub-functions are working as expected.

The big picture of simulation studies
-------------------------------------

In a typical data analysis setting, there are population parameters that one hopes to estimate by collecting and analyzing data. The population parameters are unknown, and the accuracy of the conclusions is unknown.

![](/simulation-tutorial_files/a.svg)

In a simulation setting, the researcher sets the population parameters then generates data using the parameters. After completing the analysis, the researcher can then evaluate the accuracy of the conclusions. If the research repeats this process several times, then she/he can estimate the long-run operating characteristics of the analysis procedure *for the specific set of population parameters*.

![](/simulation-tutorial_files/b.svg)

The framework described above suggests how one may write modular code to perform the simulation. One can write a function to perform each of the primary tasks. For example:

![](/simulation-tutorial_files/c.svg)

Example A
---------

Consider the example of a clinical trial comparing length-of-stay between a new surgical approach and a traditional surgical approach. With the traditional surgical approach, length-of-stay follows a mixture distribution. One fourth of the patients stay zero days in the hospital. The other portion have a length-of-stay distribution that follows a discretized, shifted-by-one exponential with a mean of 4 days.
The researchers believe the new approach will not alter the number of subjects that will stay zero nights. Rather, they believe that it will reduce the length-of-stay among patients that stay at least one night.

The primary endpoint will be analyzed with a Wilcoxon two sample test.

Researchers want to know the minimal detectable difference at 80% power if they enroll 150 patients in each arm.

``` r
require(magrittr, quietly = TRUE)

parameters <- c(N = 150, trt_effect = 1)

generate_data <- function(parameters){
  N <- parameters[1]
  trt_effect <- parameters[2]
  trt <- rep(0:1, each = N)
  G <- rbinom(2*N, 1, p = 0.75)
  means <- 4 - 1 - trt_effect*trt
  Y_tmp <- 1 + rexp(2*N, 1/means) %>% floor
  Y <- Y_tmp*G + 0*(1-G)
  
  data.frame(Y = Y, trt = trt)
}

analysis <- function(D, alpha = 0.05){
  wilcox.test(Y~trt, data = D)$p.value < alpha
}

# This function isn't needed for this particular simulation, 
# but it might be helpful for the simulation in the qualifying exam.
oper_char <- function(x) 1*x

one_rep <- function(parameters) parameters %>% generate_data %>% analysis %>% oper_char
# Note that sim_params = c(M, parameters)
M_rep <- function(sim_params) replicate(sim_params[1], one_rep(sim_params[2:3]))
```

### Simulation settings and simulation

``` r
ss1 <- expand.grid(
    M = 1e4
  , N = 150
  , trt_effect = seq(0, 2, by = 1/3) %>% round(2)
  , power = NA
)

# Uses previous results if available
if("simulation-results.rds" %in% list.files()){
  ss1 <- readRDS("simulation-results.rds")
}else{
  for(i in 1:nrow(ss1)){ss1[i,4] <- ss1[i,1:3] %>% as.numeric %>% M_rep %>% mean}  
  saveRDS(ss1, "simulation-results.rds")
}
```

### Results

``` r
ss1 %>% kable
```

|      M|    N|  trt\_effect|   power|
|------:|----:|------------:|-------:|
|  10000|  150|         0.00|  0.0524|
|  10000|  150|         0.33|  0.0751|
|  10000|  150|         0.67|  0.1838|
|  10000|  150|         1.00|  0.3935|
|  10000|  150|         1.33|  0.6565|
|  10000|  150|         1.67|  0.8970|
|  10000|  150|         2.00|  0.9846|

### Power plot

``` r
tgsify::plotstyle(style = upright)
plot(ss1$trt_effect, ss1$power, xlab = "Treatment effect", ylab = "Power")
lm1 <- glm(power ~ poly(trt_effect, degree = 3, raw = TRUE), data = ss1, family = binomial)
xx <- seq(0,2,by=.1)
power_smooth <- predict(lm1, newdata = data.frame(trt_effect = xx), type = "response")
lines(xx, power_smooth, lwd = 3)
abline(h = 0.8, col = "grey")
(power_smooth - 0.8) %>% abs %>% which.min %>% `[`(xx, .) %>% abline(v = ., col = "grey")
```

![](simulation-tutorial_files/figure-markdown_github/unnamed-chunk-7-1.svg)

Example B
---------

A pharmaceutical company has been commissioned to rapidly develop a new antibiotic that will be more effective against a specific bacterial infection that is highly resistant to standard therapy. To speed up development the research team has decided to simultaneously test 10 new candidate medications against standard therapy in a randomized controlled trial. This has created methodological challenges the biostatistical team is working to resolve.

They have agreed on a study design that will collect the study outcome (infection resolved or not resolved within 14 days) on 400-500 patients for the standard therapy (control arm) and 50-100 patients for each of the 10 candidate therapies (intervention arms). For sake of estimating the operational characteristics of the proposed analysis approaches, they assume each of the sample sizes will be uniformly distributed within those ranges. They will also assume all patients' outcomes will be independent of each other and the probability of resolution under standard therapy is 0.10.

A group of biostatisticians (ONE) is concerned about controlling for the familywise Type I error (FWER) for testing the 10 treatments against the control group all at once. They propose first performing a single chi-square test on all 11 groups to test for any differences. This test would be evaluated at a 5% significance level. If it was not significant, they would conclude there were no differences in resolution rates. If it was significant, they would then individually test each of the treatments against the control with a chi-square test each at a 5% significance level. Any of those tests that returned p- values &lt; 0.05 would be deemed statistically significant.

Another group of biostatisticians (TWO) is concerned about controlling for the FWER without losing too much Power. They propose going straight to individually testing each of the treatments against the control with chi-square tests, but doing a Bonferroni correction to maintain a 5% familywise error rate. Any of those tests that returned p- values less than this threshold would be deemed statistically significant.

A third group of biostatisticians (THREE) is concerned about not focusing on what is clinically meaningful and meaningless. They ask the researchers what a clinically meaningful improvement would be. The researchers reply around a 15% absolute gain, e.g. if the control had a 10% resolution rate, a treatment with a 25% rate would be truly meaningful. Then the biostatisticians ask what would be a clinically trivial change from the standard therapy. The team replies anything within a 2% absolute difference, e.g. going from 10% to 12%, would be trivial -- essentially meaningless. Group THREE proposes calculating the 95% confidence intervals for the risk difference for each of the 10 interventions vs the control, and deeming any interval that falls above a 2% absolute improvement to be clinically nontrivial, i.e. when the CI has a lower bound &gt; 0.02.

For the tests, all the biostatisticians agree on using the default chi-square test function in R, chisq.test(), with the default function settings. For the risk difference confidence intervals, they agree to use the Agresti-Caffo interval as implemented by the wald2ci() function in the R package PropCIs.

1.  Write a brief paragraph comparing the familywise Type I error rates (FWER) of the three analysis methods (Proposals ONE, TWO, and THREE) for testing the 10 new therapies versus the control therapy. Report your error rates to three decimal places and design your methods so you have a high degree of certainty on the first two decimal places. <br><br> Write a second paragraph describing your methods for estimating these error rates in detail and provide supporting code and figures as appropriate. – Note: Clearly explaining what you were trying to do is very important if it turns out you have an error in your code. It helps the graders give you partial credit.
    <blockquote>
    <ul>
    <li>
    Write the functions <tt>generate\_data()</tt>, <tt>analysis1()</tt>, <tt>analysis2()</tt>, <tt>analysis3()</tt>, <tt>fwer()</tt>.
    <li>
    Write <tt>generate\_data()</tt> so that one of the inputs is a vector of event probabilities corresponding to each arm of the study.
    <li>
    Write the analysis functions in a way that the input and output is exactly the same.
    <li>
    Write <tt>fwer()</tt> in a way so that it calculates the fwer regardless of the analysis method.
    <li>
    Write a <tt>one\_rep()</tt> function so that each generated dataset is analyzed by all three methods. Perhaps the output is a triple.
    </ul>
    </blockquote>
2.  Write a brief paragraph offering intuition on why each the three methods behaved the way they did in part a. Attempt to explain why each method either 1) achieved a FWER of 5% exactly, 2) was conservative by achieving a FWER less than 5%, or 3) failed to keep FWER under 5%.

3.  Write a brief paragraph comparing the Power of the three analysis methods (Proposals ONE, TWO, and THREE) for detecting that one new treatment, call it treatment A, is different than the control therapy assuming treatment A has a resolution rate of 26% and all the other therapies retain the control rate of 10%. Here Power is referring to the probability of detecting the effect of treatment A specifically. Missing the effect of A and finding a false positive effect in a different therapy doesn't count as a successful study. Report your Power estimates to three decimal places and design your methods so you have a high degree of certainty on the first two decimal places.<br> <br> Write a second paragraph describing your methods for estimating the Power in detail and provide supporting code and figures as appropriate.
    <blockquote>
    You should be able to complete this step by writing 1 additional function and reusing the code from part (a).
    </blockquote>
4.  Write a brief paragraph commenting on which method had the best Power out of the methods that controlled FWER at 5% or less. Attempt to offer insight into why the best performing method outperformed the others.

5.  Assume that the settings for parts a and c were the only two scenarios that could occur with non-negligible probability and that they were equally likely to happen. Write a brief paragraph commenting on the False Discovery and False Non-Discovery Rates of the three approaches.<br> <br> Write a second paragraph describing your methods for estimating the FDR and FNR in detail and provide supporting code and figures as appropriate. After the study was done and it was being analyzed, analysts wouldn’t omnisciently know there is at most one real effect like we are assuming here. So it is important to allow the methods to potentially find more than one significant result in a given study.<br>
    <blockquote>
    You'll need to write an additional function for this step. It will also require you two output more than a single number from the <tt>oper\_char()</tt> function.
    </blockquote>
6.  Explain why the FDR and FNR are of interest to the statisticians designing the study. Attempt to offer insight into why the best performing method outperformed the others.
