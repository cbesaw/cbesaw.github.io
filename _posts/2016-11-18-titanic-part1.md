---
title: 'Predicting Survival on the Titanic: Part 1 Logistic Regression'
output: html_document
layout: post
author: Clayton Besaw
comments: true
---

This post chronicles my attempt at the Kaggle Titanic competition. The objective of this competition is to build a classifier that correctly predicts passenger survival in the training data. My high score as a result of this effort is 370th out of ~5800, or roughly the top 7%. This post is only part 1 of this process. It outlines the use of logistic regression as a classifier and model specification. Part 2 will detail the random forest approach, which had much better results. 


#### Data Munging and Variable Inspection


First, import the CSV data anyway you like.






{% highlight r %}
train <- read.csv(".../train.csv")

test <- read.csv(".../test.csv")
{% endhighlight %}

For this process as well, let's combine the training and test sets for easier data manipulation. The test set starts at passenger 892, so we can resplit the data later for analysis.


{% highlight r %}
test$Survived <- NA #test needs matching column length for binding

comb_df <- rbind(train, test)
{% endhighlight %}

Use the head() function to inspect the first six observations for each variable. Also, the str() function is useful for looking at data-frame component characteristics. These include data type, measurement levels, and matching variable names for this information.  




{% highlight r %}
head(comb_df)
{% endhighlight %}



{% highlight text %}
##   PassengerId Survived Pclass
## 1           1        0      3
## 2           2        1      1
## 3           3        1      3
## 4           4        1      1
## 5           5        0      3
## 6           6        0      3
##                                                  Name    Sex Age SibSp
## 1                             Braund, Mr. Owen Harris   male  22     1
## 2 Cumings, Mrs. John Bradley (Florence Briggs Thayer) female  38     1
## 3                              Heikkinen, Miss. Laina female  26     0
## 4        Futrelle, Mrs. Jacques Heath (Lily May Peel) female  35     1
## 5                            Allen, Mr. William Henry   male  35     0
## 6                                    Moran, Mr. James   male  NA     0
##   Parch           Ticket    Fare Cabin Embarked
## 1     0        A/5 21171  7.2500              S
## 2     0         PC 17599 71.2833   C85        C
## 3     0 STON/O2. 3101282  7.9250              S
## 4     0           113803 53.1000  C123        S
## 5     0           373450  8.0500              S
## 6     0           330877  8.4583              Q
{% endhighlight %}



{% highlight r %}
str(comb_df)
{% endhighlight %}



{% highlight text %}
## 'data.frame':	1309 obs. of  12 variables:
##  $ PassengerId: int  1 2 3 4 5 6 7 8 9 10 ...
##  $ Survived   : int  0 1 1 1 0 0 0 0 1 1 ...
##  $ Pclass     : int  3 1 3 1 3 3 1 3 3 2 ...
##  $ Name       : Factor w/ 1307 levels "Abbing, Mr. Anthony",..: 109 191 358 277 16 559 520 629 417 581 ...
##  $ Sex        : Factor w/ 2 levels "female","male": 2 1 1 1 2 2 2 2 1 1 ...
##  $ Age        : num  22 38 26 35 35 NA 54 2 27 14 ...
##  $ SibSp      : int  1 1 0 1 0 0 0 3 0 1 ...
##  $ Parch      : int  0 0 0 0 0 0 0 1 2 0 ...
##  $ Ticket     : Factor w/ 929 levels "110152","110413",..: 524 597 670 50 473 276 86 396 345 133 ...
##  $ Fare       : num  7.25 71.28 7.92 53.1 8.05 ...
##  $ Cabin      : Factor w/ 187 levels "","A10","A14",..: 1 83 1 57 1 1 131 1 1 1 ...
##  $ Embarked   : Factor w/ 3 levels "C","Q","S": 3 1 3 3 3 2 3 3 3 1 ...
{% endhighlight %}


One should quickly notice that the the dependent variable (survived) is not a factor. Pclass also appears to be misclassified as an integer as well. These should be transformed into the factor data type with ordered levels. I make new variables for each factor because I prefer leaving the original variables in their original form. 


{% highlight r %}
comb_df$Survived2 <- as.factor(comb_df$Survived)

comb_df$Pclass2 <- as.factor(comb_df$Pclass)
{% endhighlight %}


Now let's examine the extent of missing data. This can be easily done by using the VIM package and the aggr plot function. Let's also quickly remove the survived variable since it is obviously missing for the test data.


{% highlight r %}
library(VIM)
{% endhighlight %}


{% highlight r %}
comb_df2 <- comb_df[ ,-c(2,13)]

aggr_plot <- aggr(comb_df2, col=c('navyblue','red'), numbers=TRUE, sortVars=TRUE, labels=names(comb_df2), cex.axis=.7, gap=3, 
                  ylab=c("Histogram of missing data","Pattern"))
{% endhighlight %}

![center](http://cbesaw.github.io/figs/titanic_post.rmd/unnamed-chunk-7-1.png)

{% highlight text %}
## 
##  Variables sorted by number of missings: 
##     Variable        Count
##          Age 0.2009167303
##         Fare 0.0007639419
##  PassengerId 0.0000000000
##       Pclass 0.0000000000
##         Name 0.0000000000
##          Sex 0.0000000000
##        SibSp 0.0000000000
##        Parch 0.0000000000
##       Ticket 0.0000000000
##        Cabin 0.0000000000
##     Embarked 0.0000000000
##      Pclass2 0.0000000000
{% endhighlight %}

The histogram and resulting table indicate that roughly 20% of age values are missing, and that a very small percent (0.0007) of the fare values are missing.[^1] Logistic regression can handle missing values through listwise deletion, but this is not a sound strategy if we have the tools to impute missing values. Methods like Random Forests can't work with missing values anyways, so we should really find a nice imputation solution. Let's assume that these values are missing at random and are of an adequately low percentage of full data-frame representation.[^2]   


Since fare is only missing one, it is easily enough to impute using some central tendency measure. The mean is 33.29, while the median is 14.45. Inpecting the fare variable shows a number of problems.


{% highlight r %}
summary(comb_df$Fare)
{% endhighlight %}



{% highlight text %}
##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max.    NA's 
##   0.000   7.896  14.450  33.300  31.280 512.300       1
{% endhighlight %}


{% highlight r %}
par(mfrow=c(1,2))

plot(comb_df$Fare)

hist(comb_df$Fare)
{% endhighlight %}

![center](http://cbesaw.github.io/figs/titanic_post.rmd/unnamed-chunk-9-1.png)


First, there are Fares of 0. Did these people get on for free? Probably not. Second, it is clear upon inspection that outliers are pushing the mean Fare value up. Now we have more than just a simple imputation to deal with. We can impute the missing Fare with the  median fare based on his class using the Rpart package, but first let's set the zero values to NA and impute median values based on class. 




{% highlight r %}
comb_df$fare2 <- ifelse(comb_df$Fare==0, NA, comb_df$Fare)

summary(comb_df$fare2) #inspect to see no more zero value, and now 18 NAs
{% endhighlight %}



{% highlight text %}
##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max.    NA's 
##   3.171   7.925  14.500  33.730  31.330 512.300      18
{% endhighlight %}


{% highlight r %}
library(rpart)

fareimp <- rpart(fare2 ~ Pclass, data=comb_df[!is.na(comb_df$fare2),], 
                method="anova")

comb_df$fare2[is.na(comb_df$fare2)] <- predict(fareimp, comb_df[is.na(comb_df$fare2),])

summary(comb_df$fare2) #inspect to see no more NAs
{% endhighlight %}



{% highlight text %}
##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
##   3.171   7.925  14.500  33.880  31.390 512.300
{% endhighlight %}

Now let's impute age based on multiple covariates.



{% highlight r %}
ageimp <- rpart(Age ~ Pclass2 + Sex + SibSp + Parch + fare2 + Embarked,
                data=comb_df[!is.na(comb_df$Age),], 
                method="anova")

comb_df$age2[is.na(comb_df$age2)] <- predict(ageimp, comb_df[is.na(comb_df$age2),])
{% endhighlight %}



{% highlight text %}
## Warning in is.na(comb_df$age2): is.na() applied to non-(list or vector) of
## type 'NULL'
{% endhighlight %}



{% highlight text %}
## Warning in is.na(comb_df$age2): is.na() applied to non-(list or vector) of
## type 'NULL'
{% endhighlight %}



{% highlight r %}
summary(comb_df$age2) #inspect to make sure.
{% endhighlight %}



{% highlight text %}
## Length  Class   Mode 
##      0   NULL   NULL
{% endhighlight %}

Finally in terms of data extraction, let's look at titles housed in the name variable. People more clever than I noticed that titles were consistent throughout the names listed in the passenger data. Historically, titles were also associated with different personal characteristics such as gender, class, nobility, etc. 




{% highlight r %}
boxplot(Fare ~ Pclass, data = train, xlab = "Passenger Class", ylab = "Fare Price", main = "Relationship between Fare and Class")
{% endhighlight %}

![center](http://cbesaw.github.io/figs/titanic_post.rmd/unnamed-chunk-14-1.png)



{% highlight r %}
train$lfare <- log(train$Fare + 1)
test$lfare <- log(test$Fare + 1)
train$iage <- with(train, impute(Age, mean))
test$iage <- with(test, impute(Age, mean))
test$ifare <- with(test, impute(Fare, mean))
test$lifare <- log(test$ifare + 1) 
train$lifare <- train$lfare
train$south <- ifelse(train$Embarked == "S", 1, 0)
train$south <- as.factor(train$south)
test$south <- ifelse(test$Embarked == "S", 1, 0)
test$south <- as.factor(test$south)
train$priority <- 0
train$priority[which(train$Sex=="female" | train$iage < 16)] <- 1
test$priority <- 0
test$priority[which(test$Sex=="female" | test$iage < 16)] <- 1
train$child <- 0
train$child[which(train$iage < 16)] <- 1
test$child <- 0
test$child[which(test$iage < 16)] <- 1
{% endhighlight %}




## Logistic Regression

Logistic regression is a classic classifier in statistics and econometrics. It is also heavily used in political science and sociology as a workhorse modeling method. Like linear regression and classification methods, we are interested in understanding the probability of an outcome $Y=1$ occurring conditional on a set of $n$ predictor variables within $x\prime$. 

$$ Pr(Y = 1|x\prime) $$

One can read the above as being the probability of $Y$ equaling 1 given the measures found in $x\prime$. To model this predictive relationship, we can utilize conventions from linear regression.

First we can represent the above conditional probability as:

$$ Pr(Y = 1|x\prime) = p(x\prime) $$

Next, propose a linear function of this probability:

$$ p(x\prime) = \beta_{0} + \beta_{i}x_{i}...+ \beta_{n}x_{n} $$

Unfortunately, stopping here would not help us. Our model would sometimes result in probabilities that are $p(x\prime) < 0$ or $p(x\prime) > 1$. It is impossible to have probabilities outside of the range  $p \in [0,1]$. Fortunately, we can use the logistic function to guarantee outputs between 0 and 1 as such:

$$ p(x\prime) = \frac{e^{\beta_{0}+\beta_{i}x_{i}...+\beta_{n}x_{n}}}{1 + e^{\beta_{0}+\beta_{i}x_{i}...+\beta_{n}x_{n}}} $$

From here we can calculate the odds of $p(x\prime$):

$$ \frac{p(x\prime)}{1 - p(x\prime)} = e^{\beta_{0} + \beta_{i}x_{i}...+\beta{n}x_{n}} $$

Reiterating basic probability, the odds tells us about the probability of $Y = 1$. Odds close to 0 indicate very low probabilities for $Y = 1$, while probabilities ever approaching $\infty$ indicate very high probabilities for $Y = 1$. By logging both sides of the above odds equation, we can calculate the logit or logged odds:

$$ log \bigg( \, \frac{p(x\prime)}{1 - p(x\prime)} \bigg) \, = \beta_{0} + \beta_{i}x_{i}...+ \beta{n}x_{n} $$

This is the basis for using logistic regression to make classification predictions. We can interpret the influence of predictor $x_{i}$ as the change in the logged odds of $Y = 1$ by $\beta_{i}$. This is not a linear increase like the linear regression interpretation, but is a conditional increase based on the current value of $x_{i}$. Even though the relationship is non-linear, we can generalize the interpretation of $\beta_{i}$. If $\beta_{i}$ is positive, then increasing the value of $x_{i}$ will increase the logged odds of $Y = 1$. As such, a negative $\beta_{i}$ would have the opposite effect. 


The above seems easy enough, but how can we know the values of $\beta_{0}$, $\beta_{i}$, etc..? These parameters are generally estimated though the maximum likelihood method. Let's assume $n$ observations and that $x$ and $\beta$ are sets of covariates and their corresponding beta estimates. First start with the likelihood function:

$$ \ell(\beta_{0},\beta) = \prod_{i = 1}^{n}p(x_{i})^{y_{i}}(1 - p(x_{i}))^{1 - y_{i}} $$

Through further derivation of the log-likelihood function, products turn into sums:

$$ \ell(\beta_{0}, \beta) = \sum_{i=1}^{n}-log1 + e^{\beta_{0} + \beta x_{i}} + \sum_{i=1}^{n}y_{i}(\beta_{0} + \beta x_{i}) $$

Finally, one can find the maximum likelihood estimates for a specific parameter by taking the partial derivative of the log likelihood function. Say we wanted the estimate $\beta_{j}$ from the set of estimates $\beta$. We take the partial derviative of the log likelihood function with respect to $\beta_{j}$

$$ \frac{\partial \ell}{\partial \beta_{j}} = -\sum_{i=1}^{n} \frac{1}{1 + e^{\beta_{0} + \beta x_{i}}} e^{\beta_{0} + \beta x_{i}}x_{ij} + \sum_{i=1}^{n} y_{i}x_{ij} $$


$$ = \sum_{i=1}^{n}(y_{i} - p(x_{i};\beta_{0},\beta))x_{ij} $$


From here, we have the basis for estimating $\beta_{j}$. The result is an approximation, as the resulting equation is transcendental. 





[^1]: It is actually just one..

[^2]: Multiple imputation assumes that missing values are missing at random (MAR). One should be careful when performing multiple imputation on real world data, as the MAR assumption is often not feasible. I am not sure why Age has so many missing values here. It could be a design by Kaggle, which is what I personally assume for this analysis. Normally, if one variable has such a high degree of missingness, it is unlikely to be missing at random. In addition, missing values should normally not be more than 5%-10% of a measure's representation. Having much higher degrees of missingness is often a sign of MAR violation. Consequently, with more missingness, you are more likely to artificially construct patterns within the imputed data if your baseline data is unrepresentative of the real population. 
