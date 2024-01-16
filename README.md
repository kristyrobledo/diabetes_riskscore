---
output: github_document
---



# Diabetes riskscore (additional files)


## Model objects (from R)

The R model objects for the three models are as follows:

- `baseline_model.Rdata` = baseline risk factors: 2-hour OGTT glucose, HbA1c and treatment with testosterone
- `baseline_weight_model.Rdata` = baseline risk factors plus baseline weight and changes in weight at one year
- `baseline_waist_model.Rdata` = baseline risk factors plus baseline waist circumference and changes in waist circumference at one year

## Names and units

### baseline_model

**Outcome**

- `ogtt_gt11` = factor. 0= "no diabetes at 2 years", 1="diabetes at 2 years", as determined by 2-hour OGTT glucose >=11.1mmol/l 

**Covariates**

- `cxhba1c` = numeric. Hba1c level at baseline in %. 
- `cxglu2h` = numeric. 2-hour OGTT glucose at baseline in mmol/L , 
- `t_x` = numeric. Interaction term with treatment and hba1c, see example for how to calculate. Briefly: if placebo, always 0. If Testosterone, and hba1c values is <5.6%, its 0. If Hba1c is >5.6%, its the amount that the hba1c value is >5.6%. 

### baseline_weight_model

Model looks like so: `outcome ~ logit + base_wt + change_wt`

**Outcome**

- `outcome` = factor. 0= "no diabetes at 2 years", 1="diabetes at 2 years", as determined by 2-hour OGTT glucose >=11.1mmol/l 

**Covariates**

- `logit` = numeric. The linear predictor from the `baseline_model`
- `base_wt` = numeric. Weight at baseline in kilograms, 
- `change_wt` = numeric. Change in weight at one year from baseline, in kilograms (ie. weight at one year minus weight at baseline)

### baseline_waist_model

Model looks like so: `outcome ~ logit + base_wc + change_wc`

**Outcome**

- `outcome` = factor. 0= "no diabetes at 2 years", 1="diabetes at 2 years", as determined by 2-hour OGTT glucose >=11.1mmol/l 

**Covariates**

- `logit` = numeric. The linear predictor from the `baseline_model`
- `base_wc` = numeric. Waist circumference at baseline in centimetres, 
- `change_wc` = numeric. Change in Waist circumference at one year from baseline, in centimetres (ie. waist at one year minus waist at baseline)

## Example of use

To use the risk score, let create some fake patients to use this on. The hardest part is to create the interaction variable - straightforward with the code below:


```r
# setup hba1c and glucose values for 10 patients
cxhba1c <- seq(min(4.1), max(6.4), length.out = 10)
cxglu2h <- seq(min(7.7), max(11), length.out = 10)

# create a dataset with 10 placebo (treat=0) and 10 testosterone (treat=1) patients 
as.data.frame(
  rbind(
    cbind( cxhba1c=cxhba1c, cxglu2h = cxglu2h, treat=rep(0,10)), 
    cbind( cxhba1c=cxhba1c, cxglu2h = cxglu2h, treat=rep(1,10))
  )
) ->pred_df

glimpse(pred_df)
```

```
#> Rows: 20
#> Columns: 3
#> $ cxhba1c [3m[38;5;246m<dbl>[39m[23m 4.100000, 4.355556, 4.611111, 4.866667, 5.122222, 5.377778, 5.633333, 5.888889~
#> $ cxglu2h [3m[38;5;246m<dbl>[39m[23m 7.700000, 8.066667, 8.433333, 8.800000, 9.166667, 9.533333, 9.900000, 10.26666~
#> $ treat   [3m[38;5;246m<dbl>[39m[23m 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1
```

```r
##create the interaction variable
pred_df %>%
  mutate(
    t_x = case_when(treat==0 ~0,
                treat==1 & cxhba1c<5.6  ~0,
                treat==1 & cxhba1c>=5.6~ cxhba1c-5.6)) ->pred_df

glimpse(pred_df)
```

```
#> Rows: 20
#> Columns: 4
#> $ cxhba1c [3m[38;5;246m<dbl>[39m[23m 4.100000, 4.355556, 4.611111, 4.866667, 5.122222, 5.377778, 5.633333, 5.888889~
#> $ cxglu2h [3m[38;5;246m<dbl>[39m[23m 7.700000, 8.066667, 8.433333, 8.800000, 9.166667, 9.533333, 9.900000, 10.26666~
#> $ treat   [3m[38;5;246m<dbl>[39m[23m 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1
#> $ t_x     [3m[38;5;246m<dbl>[39m[23m 0.00000000, 0.00000000, 0.00000000, 0.00000000, 0.00000000, 0.00000000, 0.0000~
```

```r
head(pred_df)
```

```
#>    cxhba1c  cxglu2h treat t_x
#> 1 4.100000 7.700000     0   0
#> 2 4.355556 8.066667     0   0
#> 3 4.611111 8.433333     0   0
#> 4 4.866667 8.800000     0   0
#> 5 5.122222 9.166667     0   0
#> 6 5.377778 9.533333     0   0
```

```r
tail(pred_df)
```

```
#>     cxhba1c   cxglu2h treat        t_x
#> 15 5.122222  9.166667     1 0.00000000
#> 16 5.377778  9.533333     1 0.00000000
#> 17 5.633333  9.900000     1 0.03333333
#> 18 5.888889 10.266667     1 0.28888889
#> 19 6.144444 10.633333     1 0.54444444
#> 20 6.400000 11.000000     1 0.80000000
```

Now we have our prediction dataset, we can calculate the probability of diabetes at 2 years for these patients (`fit`, and the standard error called `fit.se`):


```r
# load the baseline model, saved in the same spot as this file
load("baseline_model.Rdata")

prob_df<-cbind(pred_df, predict(baseline_model,
                                type="response",
                                se.fit=TRUE,
                                newdata = pred_df))

head(prob_df)
```

```
#>    cxhba1c  cxglu2h treat t_x          fit       se.fit residual.scale
#> 1 4.100000 7.700000     0   0 0.0002075621 0.0001901259              1
#> 2 4.355556 8.066667     0   0 0.0006490956 0.0005016946              1
#> 3 4.611111 8.433333     0   0 0.0020279698 0.0012790002              1
#> 4 4.866667 8.800000     0   0 0.0063174702 0.0030945804              1
#> 5 5.122222 9.166667     0   0 0.0195026839 0.0068699586              1
#> 6 5.377778 9.533333     0   0 0.0585843755 0.0132320206              1
```

```r
tail(prob_df)
```

```
#>     cxhba1c   cxglu2h treat        t_x        fit      se.fit residual.scale
#> 15 5.122222  9.166667     1 0.00000000 0.01950268 0.006869959              1
#> 16 5.377778  9.533333     1 0.00000000 0.05858438 0.013232021              1
#> 17 5.633333  9.900000     1 0.03333333 0.15223627 0.021191064              1
#> 18 5.888889 10.266667     1 0.28888889 0.23212145 0.038627900              1
#> 19 6.144444 10.633333     1 0.54444444 0.33724998 0.084164814              1
#> 20 6.400000 11.000000     1 0.80000000 0.46138304 0.138685448              1
```

To apply to the models that include changes in body composition, we follow the same process. 
First lets create our dataset of patients:


```r
cxhba1c <- seq(min(4.1), max(6.4), length.out = 10)
cxglu2h <- seq(min(7.7), max(11), length.out = 10)
base_wt <- seq(min(65), max(170), length.out = 10)
change_wt <- seq(min(-50), max(16), length.out = 10)

# create a dataset with 10 placebo (treat=0) and 10 testosterone (treat=1) patients 
as.data.frame(
  rbind(
    cbind( cxhba1c=cxhba1c, cxglu2h = cxglu2h, 
           base_wt=base_wt,change_wt=change_wt, treat=rep(0,10)), 
    cbind( cxhba1c=cxhba1c, cxglu2h = cxglu2h,
           base_wt=base_wt,change_wt=change_wt, treat=rep(1,10))
  )
) ->predch_df

glimpse(predch_df)
```

```
#> Rows: 20
#> Columns: 5
#> $ cxhba1c   [3m[38;5;246m<dbl>[39m[23m 4.100000, 4.355556, 4.611111, 4.866667, 5.122222, 5.377778, 5.633333, 5.8888~
#> $ cxglu2h   [3m[38;5;246m<dbl>[39m[23m 7.700000, 8.066667, 8.433333, 8.800000, 9.166667, 9.533333, 9.900000, 10.266~
#> $ base_wt   [3m[38;5;246m<dbl>[39m[23m 65.00000, 76.66667, 88.33333, 100.00000, 111.66667, 123.33333, 135.00000, 14~
#> $ change_wt [3m[38;5;246m<dbl>[39m[23m -50.000000, -42.666667, -35.333333, -28.000000, -20.666667, -13.333333, -6.0~
#> $ treat     [3m[38;5;246m<dbl>[39m[23m 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1
```

```r
##create the interaction variable
predch_df %>%
  mutate(
    t_x = case_when(treat==0 ~0,
                treat==1 & cxhba1c<5.6  ~0,
                treat==1 & cxhba1c>=5.6~ cxhba1c-5.6)) ->predch_df

glimpse(predch_df)
```

```
#> Rows: 20
#> Columns: 6
#> $ cxhba1c   [3m[38;5;246m<dbl>[39m[23m 4.100000, 4.355556, 4.611111, 4.866667, 5.122222, 5.377778, 5.633333, 5.8888~
#> $ cxglu2h   [3m[38;5;246m<dbl>[39m[23m 7.700000, 8.066667, 8.433333, 8.800000, 9.166667, 9.533333, 9.900000, 10.266~
#> $ base_wt   [3m[38;5;246m<dbl>[39m[23m 65.00000, 76.66667, 88.33333, 100.00000, 111.66667, 123.33333, 135.00000, 14~
#> $ change_wt [3m[38;5;246m<dbl>[39m[23m -50.000000, -42.666667, -35.333333, -28.000000, -20.666667, -13.333333, -6.0~
#> $ treat     [3m[38;5;246m<dbl>[39m[23m 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1
#> $ t_x       [3m[38;5;246m<dbl>[39m[23m 0.00000000, 0.00000000, 0.00000000, 0.00000000, 0.00000000, 0.00000000, 0.00~
```

```r
head(predch_df)
```

```
#>    cxhba1c  cxglu2h   base_wt change_wt treat t_x
#> 1 4.100000 7.700000  65.00000 -50.00000     0   0
#> 2 4.355556 8.066667  76.66667 -42.66667     0   0
#> 3 4.611111 8.433333  88.33333 -35.33333     0   0
#> 4 4.866667 8.800000 100.00000 -28.00000     0   0
#> 5 5.122222 9.166667 111.66667 -20.66667     0   0
#> 6 5.377778 9.533333 123.33333 -13.33333     0   0
```

```r
tail(predch_df)
```

```
#>     cxhba1c   cxglu2h  base_wt  change_wt treat        t_x
#> 15 5.122222  9.166667 111.6667 -20.666667     1 0.00000000
#> 16 5.377778  9.533333 123.3333 -13.333333     1 0.00000000
#> 17 5.633333  9.900000 135.0000  -6.000000     1 0.03333333
#> 18 5.888889 10.266667 146.6667   1.333333     1 0.28888889
#> 19 6.144444 10.633333 158.3333   8.666667     1 0.54444444
#> 20 6.400000 11.000000 170.0000  16.000000     1 0.80000000
```

Now we have our prediction dataset, we can calculate the linear predictor for the baseline model, using the same process as before. This gives `fit` which is our logit:


```r
predch_df_logit<-cbind(predch_df, 
                       logit = predict(baseline_model,
                                       newdata = predch_df))
```

Now we can perform the prediction for the baseline model with weight included:


```r
load("baseline_weight_model.Rdata")

probch_df<-cbind(predch_df_logit, 
                 predict(baseline_weight_model,
                                    type="response",
                                    se.fit=TRUE,
                                    newdata = predch_df_logit))
head(probch_df)
```

```
#>    cxhba1c  cxglu2h   base_wt change_wt treat t_x     logit          fit       se.fit
#> 1 4.100000 7.700000  65.00000 -50.00000     0   0 -8.479872 2.852252e-08 6.154385e-08
#> 2 4.355556 8.066667  76.66667 -42.66667     0   0 -7.339281 3.564813e-07 6.491108e-07
#> 3 4.611111 8.433333  88.33333 -35.33333     0   0 -6.198690 4.455372e-06 6.619967e-06
#> 4 4.866667 8.800000 100.00000 -28.00000     0   0 -5.058099 5.568148e-05 6.426344e-05
#> 5 5.122222 9.166667 111.66667 -20.66667     0   0 -3.917508 6.954754e-04 5.768298e-04
#> 6 5.377778 9.533333 123.33333 -13.33333     0   0 -2.776917 8.623264e-03 4.506473e-03
#>   residual.scale
#> 1              1
#> 2              1
#> 3              1
#> 4              1
#> 5              1
#> 6              1
```

```r
tail(probch_df)
```

```
#>     cxhba1c   cxglu2h  base_wt  change_wt treat        t_x      logit          fit       se.fit
#> 15 5.122222  9.166667 111.6667 -20.666667     1 0.00000000 -3.9175078 0.0006954754 0.0005768298
#> 16 5.377778  9.533333 123.3333 -13.333333     1 0.00000000 -2.7769167 0.0086232638 0.0045064734
#> 17 5.633333  9.900000 135.0000  -6.000000     1 0.03333333 -1.7171682 0.0908307228 0.0261448097
#> 18 5.888889 10.266667 146.6667   1.333333     1 0.28888889 -1.1963708 0.3951513547 0.0877907940
#> 19 6.144444 10.633333 158.3333   8.666667     1 0.54444444 -0.6755735 0.8103237123 0.0914695044
#> 20 6.400000 11.000000 170.0000  16.000000     1 0.80000000 -0.1547761 0.9654417436 0.0290473173
#>    residual.scale
#> 15              1
#> 16              1
#> 17              1
#> 18              1
#> 19              1
#> 20              1
```





