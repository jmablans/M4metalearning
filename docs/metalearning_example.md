Forecasting Metalearning Example
================
Pablo Montero-Manso
2018-06-09

Intro
=====

The purpose of this package is to apply metalearning to time series forecasting, with a focus on the M4 competition dataset. By metalearning, we mean a method that learns to select or combine individual forecasting methods to produce more accurate forecasts. A metalearning model trained on the M4 dataset will be provided, one that can be applied directly to a given time series. Additionally, if we will provide the tools to train new metalearning models on arbitrary datasets of time series.

We will see an example of the proposed metalearning method, applied to a subset of the M4 competition dataset. We will consider one subset as the 'training set' and fit the metalearning model to it. We will test this new model on a disjoint subset of the M4 dataset and see the differences in forecasting error between using our metalearning approach, using the best single forecasting method, and using other ensemble methods.

Overview of the approach
========================

Our approach consist in training a model that predicts how to combine the forecast methods in a pool of methods in order produce a more accurate forecast. A set of features is extracted from the series we want to forecast, and these features are used as input to the metalearning model. The metalearning model will produce a weight for each forecasting method. These weights can also be interpreted as the probability of that method being the best. These weights can therefore be used to either select the best method for that series, by choosing the one with more probability, or to combine all methods by a weighted linear combination. In our experiments, the linear combination produces smaller forecasting error in general.

In order to train a metalearning model, we need a dataset of time series and a pool of forecasting methods. The forecasting methods in the pool will be applied to each time series and their forecasting error will be measured. We need a valid target for the forecasts if we want to measure the error, this can either be additionally provided in the dataset of time series, or extrated from it by applying temporal holdout: the last few observations of each series will be removed from it and considered as the true target of the forecast from the remaninig observations of the series. A set of features is extracted from each series and finally a model similar to a classifier is trained based on these features to assign probabilities to the methods in order to minimize the average error.

Procedure
=========

We start with a dataset that contains the time series of the training set. The metalearning model is trained based on the errors of the forecasts methods in the pool and the extracted features. The errors must be calculated using a ground truth for the forecasts for each series.

The ground truth may be available, as the observations that follow a given series, or a given forecast horizon may be available for each series, as in the M4 competition, or no additional information may be available. In the first scenario, the forecasting methods calculate the predictions to a time horizon of the length of the ground truth part of the series. In the second scenario, the last observations coinciding with the forecasting horizon are removed from the series and considered as ground truth, and then the procedure for the previous scenario is applied. In the last scenario, the user may specifiy a forecasting horizon, or a default of twice the frequency of the series with a minimum of six observations will be considered, and then the procedure of the second scenario will be applied.

We consider the M4 competition, so we are in the second scenario: we know the forecasting horizon. The function `temp_holdout` will create the new version of the dataset by removing the last observations of each series and set them as the ground truth. `temp_holdout` receives as input a `list` with each element having at least following fields:

-   `x` : Containing the series as a `ts` object.
-   `h` : An integer with the amount of future moments to forecast.

The output of `temp_holdout` is a `list` with the same format as the input, keeping additional fields that are not `x` or `h` and adding or overwriting the field `xx`, containing the extracted observations the each series, what will be the ground truth.

Input and output list formats are inspired by the one used in `Mcomp` and `M4comp2018` packages.

Note that we use very small train and test sample sizes, computings time for forecast calculations are large.

``` r
library(M4metalearning)
library(M4comp2018)
set.seed(31-05-2018)
#we start by creating the training and test subsets
indices <- sample(length(M4))
M4_train <- M4[ indices[1:15]]
M4_test <- M4[indices[16:25]]
#we create the temporal holdout version of the training and test sets
M4_train <- temp_holdout(M4_train)
M4_test <- temp_holdout(M4_test)
```

We then apply the forecasting methods to each series in the training dataset and keep their forecast, as well as calculate the forecasting error they produce. We measure the forecasting error using the OWI formula used in the M4 competition, an average of MASE and SMAPE errors.

This package provides the function `calc_forecasts` that takes as input the training dataset with the time series and a list of forecasting methods to apply to it. We can see in the documentation of the function `calc_forecasts` that the input dataset requires the following format: A `list` with each element having the following structure:

-   `x` : Containing the series as a `ts` object.
-   `h` : An integer with the amount of future moments to forecast.

The second parameter, `methods`, is a list of strings, each string has to be the name of an existing **`R`** function that will produce the forecast. The functions pointed by `methods` must have the following structure:

    my_forect_method <- fuction(x, h) { ... }

With x being a `ts` object to forecast and `h` the amount of future moments to forecast. **The output of these functions must be a numeric vector of length `h` containing the forecasts of the method.** This input format is used for simplicity, any parameters used by the methods must be set internally in these functions.

A list of methods wrapped from the `forecast` package is also provided in the function `forec_method_list()`. The methods in `forec_methods_list()` are the ones used in our submission to the M4 Competition.

Without further ado, lets process the training set to generate all forecast and errors, to be used in the metalearning.

``` r
#this will take time
M4_train <- calc_forecasts(M4_train, forec_methods(), n.cores=3)
#once we have the forecasts, we can calculate the errors
M4_train <- calc_errors(M4_train)
```

Now we have in `M4_train` the series, the forecasts of each method for each series, and the errors of these forecasts. Specifically, `calc_forecasts` produces as output a list, with each element having the following structure:

-   x : The series to be forecasted.
-   h : The amount of moments into the future to forecast.
-   ff : The forecasts of each methods, a matrix with the forecast of each methods per row.

The function `calc_errors` also produces a list of series, adding or overwriting the following field:

-   errors : A vector with the OWI errors produced by each method.

Feature extraction
------------------

The information in `M4_train` (the series, the forecasts and the errors produced by the forecasts) is enough to train a metalearning method, but many approaches will require further processing, e.g. extracting features, zero padding the series so all have the same length, etc.

The metalearning method we are proposing uses features extracted from the series instead of the raw series. With these features, it trains a xgboost model that gives weights to the forecast methods in order to minimize the OWI error (roughly speaking).

The specific features extracted from the series are a key part of the process, and in this package, the function `THA_features` calculates such features. A rationale and description of the features may be seen in a --upcoming paper from (Talagala, Hyndman and Athanasopoulos, 2018)--, though the actual set of features extracted is slightly different.

`THA_features` processes once again a dataset of times series, a list of elements containing at least the field `x` with a `ts` object (in the spirit of the input and output datasets of `calc_forecasts` and the `Mcomp` package data format).

The output of `THA_features` is its input list, but to each element, the field `features` has been added. `features` is a `tibble` object with the extracted features. `THA_features` uses the package `tsfeatures` for extracting the features.

The `features` field is used by other functions in the package, so users may use their own set of features by setting this field.

``` r
M4_train <- THA_features(M4_train)
#> Loading required package: tsfeatures
```

Now we have in `M4_train` the series, its extracted features, forecasts and errors.

The next step is training the ensemble using xgboost the extracted features and the error produced by each method.

The `create_feat_classif_problem` auxiliary function is provided to reformat the list produced by the previously shown functions to the more common format of a feature matrix and target labels used in classification functions such as `randomForest`, `svm`, `xgboost`, ... It uses the `features` and `errors` fields of each element in the input list and turns the into two matrices, `data` and `errors` one series per row. Columns in `data` have the features and in `errors` the error produced by each forecasting method that was applied to the dataset. Additionaly, the vector `labels` is generated by selecting which method produces the least error in each row.

`create_feat_classif_problem` simply produces a list with the entries:

-   data : A matrix with the features extracted from the series
-   errors : A matrix with the errors produced by the forecasting methods
-   labels : Integer from 0 to (number of forecasting methods - 1). The target classification problem, created by selecting the method that produces the smallest error for each series.

``` r
train_data <- create_feat_classif_problem(M4_train)
round(head(train_data$data, n=3),2)
#>      x_acf1 x_acf10 diff1_acf1 diff1_acf10 diff2_acf1 diff2_acf10
#> [1,]   0.98    7.74      -0.18        0.17      -0.58        0.51
#> [2,]   0.88    2.34       0.03        0.26      -0.48        0.54
#> [3,]   0.94    3.09       0.64        0.78       0.37        0.35
#>      seas_acf1 ARCH.LM crossing_points entropy flat_spots arch_acf
#> [1,]      0.92    0.99               1    0.47         29     0.04
#> [2,]      0.00    1.00               3    0.71          4     0.58
#> [3,]      0.10    0.93               5    0.65          5     0.30
#>      garch_acf arch_r2 garch_r2 alpha beta hurst lumpiness nonlinearity
#> [1,]      0.05    0.16     0.16  0.61 0.14  1.00      0.00         0.62
#> [2,]      0.26    1.00     1.00  1.00 0.01  0.99      0.01         2.13
#> [3,]      0.27    0.33     0.34  1.00 1.00  0.99      0.03         0.13
#>      x_pacf5 diff1x_pacf5 diff2x_pacf5 seas_pacf nperiods seasonal_period
#> [1,]    0.97         0.13         0.79      0.00        1               4
#> [2,]    0.82         0.14         0.44      0.00        0               1
#> [3,]    1.53         0.79         0.45     -0.08        1              12
#>      trend spike linearity curvature e_acf1 e_acf10 seasonal_strength peak
#> [1,]  1.00     0     10.01     -0.79  -0.31    0.40              0.20    4
#> [2,]  0.98     0      4.52     -0.29   0.33    0.70              0.00    0
#> [3,]  0.84     0     -3.55      3.02   0.63    0.65              0.37    3
#>      trough stability hw_alpha hw_beta hw_gamma unitroot_kpss unitroot_pp
#> [1,]      3      1.03     0.54    0.16        0          2.15       -0.56
#> [2,]      0      1.27     0.00    0.00        0          0.84       -0.77
#> [3,]     10      0.93     1.00    0.02        0          0.62       -5.51
#>      series_length
#> [1,]           104
#> [2,]            23
#> [3,]            51
round(head(train_data$errors, n=3),2)
#>      auto_arima_forec ets_forec nnetar_forec tbats_forec stlm_ar_forec
#> [1,]             0.14      0.19         0.29        0.19          0.56
#> [2,]             0.93      0.89         1.97        1.00          1.96
#> [3,]             1.59      1.46         2.21        1.52          1.37
#>      rw_drift_forec thetaf_forec naive_forec snaive_forec
#> [1,]           0.18         0.12        0.24         0.31
#> [2,]           0.93         1.29        1.80         1.80
#> [3,]           1.53         1.59        1.46         1.23
round(head(train_data$labels, n=3),2)
#> [1] 6 1 8
```

The data in this format is easy to use with any classifier, as we will see.

Training the learning model
---------------------------

*NOTE: At the moment, the `R` version of the `xgboost` tool does not allow simple introduction of custom loss functions for multiclass problems. A workaround is available as package in github <https://github.com/pmontman/customxgboost> and `M4metalearning` checks if this special version is installed.*

The next step is training the metalearning model: a 'learning models' that given the extracted features of the series we want to forecast, gives weights to the individual forecasting methods so that their weighted linear combination has less error on average.

One of the proposals in this package is `train_selection_ensemble`, that will create a `xgboost` model,trained with a custom objective function that requires the whole errors information instead of only which method is the best for each series. *NOTE: An important part of the process is finding the hyperparameters of the model, `train_selection_ensemble` uses the set of hyperparamets found for the M4 competition. Additional functions are included in the package that perform automatic hyperparameter search.*

``` r
set.seed(1345) #set the seed because xgboost is random!
meta_model <- train_selection_ensemble(train_data$data, train_data$errors)
```

Now we have in `meta_model` a `xgb.Booster` object that can be directly used and examined, but easy to use functions for prediction and performance measurement are also provided in the package.

Testing the model
-----------------

It only remains now to test this model with the test dataset. If we want to do prediction in a 'real' forecasting exercise, we only need to extract the features from the test dataset and process the forecasts methods, no error measure is necessary. We proceed to calculate the individual forecasts and extrat the features on the series of the test dataset.

``` r
M4_test <- calc_forecasts(M4_test, forec_methods(), n.cores=1)
#> Warning in forecast::auto.arima(x, stepwise = FALSE, approximation =
#> FALSE): Having 3 or more differencing operations is not recommended. Please
#> consider reducing the total number of differences.
M4_test <- THA_features(M4_test, n.cores=1)
```

With this new information, the forecast based on the metalearning model can be calculated. Predictions can be calculated directly using the `xgb.Booster` object and the `predict` function, but the output then must be transformed to probabilities applying the softmax transform. The function `predict_selection_ensemble` is provided in the `M4metalearning` package for ease of use. Similar to the `predict` fuctions, it takes as parameters the model and a matrix with the features of the series. It outputs the predictions of the metalearning model, a matrix with the weights of the linear combination of methods, one row for each series. In order to create the `newdata` matrix required, the function `create_feat_classif_problem` can be used, it just produces the `data` object, not `errors` and `labels`.

``` r
test_data <- create_feat_classif_problem(M4_test)
preds <- predict_selection_ensemble(meta_model, test_data$data)
head(preds)
#>            [,1]       [,2]        [,3]       [,4]        [,5]       [,6]
#> [1,] 0.03474122 0.60048158 0.017065506 0.02545097 0.009365029 0.02612570
#> [2,] 0.03935903 0.48207004 0.019333853 0.02883392 0.010609829 0.02959833
#> [3,] 0.01381138 0.08823971 0.006784393 0.01011804 0.003723068 0.01038628
#> [4,] 0.01922602 0.17241768 0.009444162 0.01408474 0.005182668 0.01445813
#> [5,] 0.04355650 0.39716590 0.021395727 0.03190893 0.011741322 0.03275487
#> [6,] 0.01601401 0.08708473 0.007866365 0.01173166 0.004316821 0.01204267
#>           [,7]       [,8]        [,9]
#> [1,] 0.2414065 0.02690370 0.018459820
#> [2,] 0.3388018 0.03047974 0.020913499
#> [3,] 0.8489029 0.01069557 0.007338703
#> [4,] 0.7400821 0.01488868 0.010215785
#> [5,] 0.4046026 0.03373027 0.023143836
#> [6,] 0.8400334 0.01240129 0.008509076
```

The last step is calculating the actual forecasts by the linear combinations produced by the metalearning model. The function `ensemble_forecast` is provided for this task. It will take the `predictions` and the `dataset` in the output of format of `calc_forecasts` with the `ff` field and add the field `y_hat` with the actual forecasts to the elements `dataset`.

``` r
M4_test <- ensemble_forecast(preds, M4_test)
M4_test[[1]]$y_hat
#>          [,1]     [,2]     [,3]     [,4]     [,5]     [,6]     [,7]
#> [1,] 7887.043 7887.986 7887.982 7887.662 7886.898 7887.026 7887.238
#>          [,8]     [,9]    [,10]    [,11]    [,12]    [,13]    [,14]
#> [1,] 7887.451 7887.666 7887.882 7888.099 7888.318 7888.539 7888.761
```

We have the forecasts!

Performance Results
-------------------

When the true targets of the forecasts are know, it is interesting to know how well the metalearning model is performing. In this example, for the test set `M4_test`, we know the 'true' future observations because it has been created artificially. Another very common scenario is crossvalidation on a training set, which part of it has been separated.

In `M4metalearning` package we provide the function `summary_performance`, that requires as input the forecast methods 'weights' or 'probabilities' predicted by the model, and the dataset with the `ff` and `xx` fields for each element.

``` r
summary <- summary_performance(preds, dataset = M4_test)
#> [1] "Classification error:  1"
#> [1] "Selected OWI :  0.8144"
#> [1] "Weighted OWI :  0.7661"
#> [1] "Naive Weighted OWI :  0.7753"
#> [1] "Oracle OWI:  0.472"
#> [1] "Single method OWI:  0.634"
#> [1] "Average OWI:  0.881"
```

The output of `summary_performance` consist of:

-   'Classification error' is the error in the classification problem of selecting the best forecasting method. In this case, by picking the method with the largest weight.
-   'Selected OWI' is the average OWI error produced by the methods selected by the classifier *(a more important measure!)*.
-   'Weighted OWI' is the error of the method created via the aforementioned combination of all individual methods by their probabilities. Either 'Selected OWI' or 'Weighted OWI" are the *real* forecasting performance measures (whichever is best), though 'Weighted' may not be available if there are restrictions on computing time. *Weighted version is the one submitted to the M4 competition*
-   'Oracle OWI' shows the theoretical minimum error that a classifier that always picks the best method for each series would produce.
-   'Single method OWI' is the erro of best method in our pool of forecasting methods.
-   'Average OWI' would be the error produced by selecting methods at random from our pool of methods for each series.
-   'Naive weighted' is the error produced by averaging the forecasts of the methods to produce a new forecast.

### Analysis of the model

Models trained with allow for some interpretation, such as relative importance of features, known as gain. We will show the top 20 features ordered by gain. Analysis tools that may be used with standard `xgboost` models could be applied to this metalearning model.

``` r
mat <- xgboost::xgb.importance (feature_names = colnames(test_data$data),
                       model = meta_model)
xgboost::xgb.plot.importance(importance_matrix = mat[1:20], cex=1.0)
```

![XGB Feature Gain of the Metalearning model](/docs/xgb_gain_plot-1.png)

### The End!

<!-- Vignettes are long form documentation commonly included in packages. Because they are part of the distribution of the package, they need to be as compact as possible. The `html_vignette` output type provides a custom style sheet (and tweaks some options) to ensure that the resulting html is as small as possible. The `html_vignette` format: -->
<!-- - Never uses retina figures -->
<!-- - Has a smaller default figure size -->
<!-- - Uses a custom CSS stylesheet instead of the default Twitter Bootstrap style -->
<!-- ## Vignette Info -->
<!-- Note the various macros within the `vignette` section of the metadata block above. These are required in order to instruct R how to build the vignette. Note that you should change the `title` field and the `\VignetteIndexEntry` to match the title of your vignette. -->
<!-- ## Styles -->
<!-- The `html_vignette` template includes a basic CSS theme. To override this theme you can specify your own CSS in the document metadata as follows: -->
<!--     output:  -->
<!--       rmarkdown::html_vignette: -->
<!--         css: mystyles.css -->
<!-- ## Figures -->
<!-- The figure sizes have been customised so that you can easily put two images side-by-side.  -->
<!-- ```{r, fig.show='hold'} -->
<!-- plot(1:10) -->
<!-- plot(10:1) -->
<!-- ``` -->
<!-- You can enable figure captions by `fig_caption: yes` in YAML: -->
<!--     output: -->
<!--       rmarkdown::html_vignette: -->
<!--         fig_caption: yes -->
<!-- Then you can use the chunk option `fig.cap = "Your figure caption."` in **knitr**. -->
<!-- ## More Examples -->
<!-- You can write math expressions, e.g. $Y = X\beta + \epsilon$, footnotes^[A footnote here.], and tables, e.g. using `knitr::kable()`. -->
<!-- ```{r, echo=FALSE, results='asis'} -->
<!-- knitr::kable(head(mtcars, 10)) -->
<!-- ``` -->
<!-- Also a quote using `>`: -->
<!-- > "He who gives up [code] safety for [code] speed deserves neither." -->
<!-- ([via](https://twitter.com/hadleywickham/status/504368538874703872)) -->