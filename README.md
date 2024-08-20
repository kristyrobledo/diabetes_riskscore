
# Diabetes riskscore (additional files)

**For the visualisation of the interaction, please see:
<https://kristyrobledo.github.io/diabetes_riskscore/>**

## Model explanation

The `baselinemodel_age.Rdata` object is an R model object for the final
model, including the following baseline risk factors: 2-hour OGTT
glucose, HbA1c, age and treatment with testosterone

**Outcome**

- `ogtt_gt11` = factor. 0= “no diabetes at 2 years”, 1=“diabetes at 2
  years”, as determined by 2-hour OGTT glucose \>=11.1mmol/l

**Covariates**

- `cxhba1c` = numeric. Hba1c level at baseline in %.
- `cxglu2h` = numeric. 2-hour OGTT glucose at baseline in mmol/L ,
- `age` = numeric. age at baseline in years (may have decimal places)
- `t_x` = numeric. Interaction term with treatment and hba1c, see
  example for how to calculate. Briefly: if planned to not have
  testosterone treatment (ie. placebo), always 0. If planning to treat
  with Testosterone, and hba1c values is \<5.6%, its 0. If Hba1c is
  \>5.6%, its the amount that the hba1c value is \>5.6%.

## Example of use

To use the risk score, let create some fake patients to use this on. The
most complex part is to create the interaction variable, but this is
straightforward with the code below:

``` r
library(tidyverse)

# setup hba1c and glucose values for 10 patients
cxhba1c <- seq(min(4.1), max(6.4), length.out = 10)
cxglu2h <- seq(min(7.7), max(11), length.out = 10)

# create a dataset with 10 placebo (treat=0) and 10 testosterone (treat=1) patients aged 59 years
as.data.frame(
  rbind(
    cbind(age=59,  cxhba1c=cxhba1c, cxglu2h = cxglu2h, treat=rep(0,10)), 
    cbind(age=59, cxhba1c=cxhba1c, cxglu2h = cxglu2h, treat=rep(1,10))
  )
) ->pred_df

#glimpse(pred_df)

##create the interaction variable
pred_df %>%
  mutate(
    t_x = case_when(treat==0 ~0,
                treat==1 & cxhba1c<5.6  ~0,
                treat==1 & cxhba1c>=5.6~ cxhba1c-5.6)) ->pred_df

#glimpse(pred_df)
head(pred_df)
```

    #>   age  cxhba1c  cxglu2h treat t_x
    #> 1  59 4.100000 7.700000     0   0
    #> 2  59 4.355556 8.066667     0   0
    #> 3  59 4.611111 8.433333     0   0
    #> 4  59 4.866667 8.800000     0   0
    #> 5  59 5.122222 9.166667     0   0
    #> 6  59 5.377778 9.533333     0   0

``` r
tail(pred_df)
```

    #>    age  cxhba1c   cxglu2h treat        t_x
    #> 15  59 5.122222  9.166667     1 0.00000000
    #> 16  59 5.377778  9.533333     1 0.00000000
    #> 17  59 5.633333  9.900000     1 0.03333333
    #> 18  59 5.888889 10.266667     1 0.28888889
    #> 19  59 6.144444 10.633333     1 0.54444444
    #> 20  59 6.400000 11.000000     1 0.80000000

Now we have our prediction dataset, we can calculate the probability of
diabetes at 2 years for these patients (`fit`, and the standard error
called `fit.se`):

``` r
# load the baseline model, saved in the same spot as this file
load("baselinemodel_age.Rdata")

prob_df<-cbind(pred_df, predict(baselinemodel_age,
                                type="response",
                                se.fit=TRUE,
                                newdata = pred_df))

head(prob_df)
```

    #>   age  cxhba1c  cxglu2h treat t_x          fit       se.fit residual.scale
    #> 1  59 4.100000 7.700000     0   0 0.0001648107 0.0001536519              1
    #> 2  59 4.355556 8.066667     0   0 0.0005371791 0.0004225419              1
    #> 3  59 4.611111 8.433333     0   0 0.0017493936 0.0011227289              1
    #> 4  59 4.866667 8.800000     0   0 0.0056815776 0.0028317379              1
    #> 5  59 5.122222 9.166667     0   0 0.0182903498 0.0065527345              1
    #> 6  59 5.377778 9.533333     0   0 0.0572693275 0.0131246395              1

``` r
tail(prob_df)
```

    #>    age  cxhba1c   cxglu2h treat        t_x        fit      se.fit residual.scale
    #> 15  59 5.122222  9.166667     1 0.00000000 0.01829035 0.006552735              1
    #> 16  59 5.377778  9.533333     1 0.00000000 0.05726933 0.013124639              1
    #> 17  59 5.633333  9.900000     1 0.03333333 0.15473813 0.021657019              1
    #> 18  59 5.888889 10.266667     1 0.28888889 0.24598067 0.040560612              1
    #> 19  59 6.144444 10.633333     1 0.54444444 0.36762532 0.088613911              1
    #> 20  59 6.400000 11.000000     1 0.80000000 0.50883144 0.141338486              1
