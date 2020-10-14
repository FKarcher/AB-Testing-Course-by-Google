# Capstone Project: A/B Testing Course by Google
1. [Udacity's A/B Testing Course by Google](#1)<br>
2. [Experiment Overview: Free Trial Screener](#2)<br>
    [2.1 Description of original conditions](#2.1)<br>
    [2.2 Description of the experimental change](#2.2)<br>
    [2.3 Experiment hypothesis](#2.3)<br>
    [2.4 How data are tracked](#2.4)<br>
3. [Metric Choice](#3)<br>
    [3.1 Choose invariant metrics](#3.1)<br>
    [3.2 Choose evaluation metrics](#3.2)<br>
4. [Measuring Standard Deviation](#4)<br>
    [4.1 Baseline values](#4.1)<br>
    [4.2 Calculate the standard deviation](#4.2)<br>
5. [Experiment Sizing](#5)<br>
    [5.1 Choose sample size](#5.1)<br>
    [5.2 Choose duration vs exposure](#5.2)<br>
6. [Experimental Analysis](#6)<br>
    [6.1 Sanity checks](#6.1)<br>
    [6.2 Effec size tests](#6.2)<br>
    [6.3 Sign tests](#6.3)<br>
7. [Conclusion and Recommendation](#7)<br>
8. [Follow-up Experiment](#8)<br>


```python
import pandas as pd
import numpy as np
import math
```

## 1. Udacity's A/B Testing Course by Google<a class="anchor" id="1"></a>
I recently finished the A/B Testing course by Google on Udacity. I highly recommend this course to people who want to learn how A/B testing is done, and to data scientists who are interested in how data science, python and statistics are applied in real-life business scenarios. The course summarized how to run an A/B test into 5 steps:
1. Choose invariant and evalution metrics.
2. Choose significance level (alpha), statistical power (1-beta) and practical significance level (the minimum change we want to observe in order to launch the change).
3. Calculate required sample size.
4. Run the test for control and experiment groups.
5. Analyze the results, calulate confidernce intervals of evaludation metrics, and draw conclusions.

This notebook is a capstone project for this course. The goal of this capstone project is to utlize the skills learned in this course and apply them to a business case using real life data. We want to gain insights about if adding an extra screening step after a student clicks the "start free trial" button will reduce the number of frustrated students who left the free trial because they couldn't commit enough hours to the course.

## 2. Experiment Overview: Free Trial Screener<a class="anchor" id="2"></a>

### 2.1 Description of original conditions<a class="anchor" id="2.1"></a>
- At the time of this experiment, Udacity currently has two options on the course overview page: "start free trial", and "access course materials". 
- If the student clicks "start free trial", they will be asked to enter their credit card information, and then they will be enrolled in a free trial for the paid version of the course. After 14 days, they will automatically be charged unless they cancel first. 
- If the student clicks "access course materials", they will be able to view the videos and take the quizzes for free, but they will not receive coaching support or a verified certificate, and they will not submit their final project for feedback.

### 2.2 Description of the experimental change<a class="anchor" id="2.2"></a>
- In the experiment, Udacity tested a change where if the student clicked "start free trial", they were asked how much time they had available to devote to the course. 
- If the student indicated 5 or more hours per week, they would be taken through the checkout process as usual. 
- If they indicated fewer than 5 hours per week, a message would appear indicating that Udacity courses usually require a greater time commitment for successful completion, and suggesting that the student might like to access the course materials for free. 
- At this point, the student would have the option to continue enrolling in the free trial, or access the course materials for free instead. This screenshot shows what the experiment looks like.
![](Experiment.png)

### 2.3 Experiment hypothesis<a class="anchor" id="2.3"></a>

The hypothesis was that this might set clearer expectations for students upfront, thus reducing the number of frustrated students who left the free trial because they didn't have enough time—without significantly reducing the number of students to continue past the free trial and eventually pay and complete the course. If this hypothesis held true, Udacity could improve the overall student experience and improve coaches' capacity to support students who are likely to complete the course.

### 2.4 How data are tracked<a class="anchor" id="2.4"></a>

**The unit of diversion is a cookie**, although if the student enrolls in the free trial, they are tracked by user-id from that point forward. The same user-id cannot enroll in the free trial twice. For users that do not enroll, their user-id is not tracked in the experiment, even if they were signed in when they visited the course overview page.

## 3. Metric Choice<a class="anchor" id="3"></a>

Which of the following metrics should we choose to measure for this experiment and why? 
For each metric we choose, we need to decide whether it is an invariant metric or an evaluation metric. The practical significance boundary for each metric, that is, the difference that would have to be observed before that was a meaningful change for the business, is given in parentheses. All practical significance boundaries are given as absolute changes.
Any place "unique cookies" are mentioned, the uniqueness is determined by day. (That is, the same cookie visiting on different days would be counted twice.) User-ids are automatically unique since the site does not allow the same user-id to enroll twice.

- Number of cookies: That is, number of unique cookies to view the course overview page. (dmin=3000)
- Number of user-ids: That is, number of users who enroll in the free trial. (dmin=50)
- Number of clicks: That is, number of unique cookies to click the "Start free trial" button (which happens before the free trial screener is trigger). (dmin=240)
- Click-through-probability: That is, number of unique cookies to click the "Start free trial" button divided by number of unique cookies to view the course overview page. (dmin=0.01)
- Gross conversion: That is, number of user-ids to complete checkout and enroll in the free trial divided by number of unique cookies to click the "Start free trial" button. (dmin= 0.01)
- Retention: That is, number of user-ids to remain enrolled past the 14-day boundary (and thus make at least one payment) divided by number of user-ids to complete checkout. (dmin=0.01)
- Net conversion: That is, number of user-ids to remain enrolled past the 14-day boundary (and thus make at least one payment) divided by the number of unique cookies to click the "Start free trial" button. (dmin= 0.0075)

### 3.1 Choose invariant metrics<a class="anchor" id="3.1"></a>

Invariant metrics are used for "sanity checks", i.e., to make sure the way we collect data for the experiment is randomized. We should pick metrics that are not affected by the experiment, so they stay the same in both our control and experiment groups.

Number of cookies, number of clicks and click-through-probability (CTR) are chosen as invariant metrics, as they are tracked BEFORE students see the experiment change; these metrics are not affected by the experimental change.

Number of user-ids are not chosen as an invariant metric, because they are only tracked AFTER users enroll in a free trial; if they are not enrolled, user-ids are not tracked.

|Invariant Metrics|Notation|Definition|Dmin|
|:---------------:|:------:|:--------:|:--:|
|# of cookies|Ck|# of unique cookies to view the course overview page|3000|
|# of clicks|Cl|# of unique cookies to click the "Start free trial" button|240|
|Click-through-probability|CTP|Cl/Ck|0.01|

### 3.2 Choose evaluation metrics<a class="anchor" id="3.2"></a>

Evaluation metrics are the metrics we expect to change, and represent practical business goals that we aim to achieve. We choose gross conversion, retention and net conversion as evaluation metrics, as they are tracked AFTER the customers click the "Start free trial" and there is a change between the control group and the experiment group.

|Evaluation Metrics|Notation|Definition|Dmin|
|:---------------:|:------:|:--------:|:--:|
|Gross conversion|GrCon|# of enrolled/Cl|0.01|
|Retention|Ret|# of paid/# of enrolled = NetCon/GrCon|0.01|
|Net conversion|NetCon|# of paid (and thus make at least one payment)/Cl|0.0075|

## 4. Measuring Standard Deviation<a class="anchor" id="4"></a>

### 4.1 Baseline values<a class="anchor" id="4.1"></a>

Udacity provided rough estimates for these metrics (the numbers have been changed from Udacity's true number); these are how the metrics behave before the change, aka, our baseline values.

|Metrics|Definition|Estimator|
|:-----:|:--------:|:-------:|
|# of cookies|Unique cookies to view course overview page per day|40000|
|# of clicks|Unique cookies to click "Start free trial" per day|3200|
|# of enrollments|Enrollments in the free trial per day|660|
|CTP|Click-through-probability on "Start free trial"|0.08|
|Gross conversion|Probability of enrollment, given click|0.20625|
|Retention|Probability of payment, given enroll|0.53|
|Net conversion|Probability of payment, given click|0.1093125|

### 4.2 Calculate the standard deviation<a class="anchor" id="4.2"></a>

For metrics we selected as our evalution metric: gross conversion, retention and net conversion, let's make an analytic estimate of its standard deviation, given **a sample size of 5000 cookies** visiting the course overview page.

First, we need to scale estimators to our sample size. In this case, from 40000 "unique cookies to view course overview page per day: to 5000 for our sample size.


```python
# Put base estimators into a dictionary
baseline = {'cookies':40000, 'clicks':3200, 'enrollments':660, 'CTP':0.08, "GrCon":0.20625, "Retention":0.53, "NetCon":0.1093125}
```


```python
# Scale estimators to our sample size
baseline2 = baseline.copy()
baseline2['cookies'] = 5000
baseline2['clicks'] = baseline['clicks']*(5000/40000)
baseline2['enrollments'] = baseline['enrollments']*(5000/40000)
baseline2
```




    {'cookies': 5000,
     'clicks': 400.0,
     'enrollments': 82.5,
     'CTP': 0.08,
     'GrCon': 0.20625,
     'Retention': 0.53,
     'NetCon': 0.1093125}



In order to estimate variances analytically, we can assume probability ($\hat {p}$) are binomially distributed; so we can use the following equation to calculate standard deviation. This equation only works when the unit of diversion of the experiment is equal to unit of the analysis (the denominator of the metric formula). In our case, both the unit of diversion and the unit of analysis are cookies.

<br>
<center><font size="4">$SD=\sqrt{\frac{\hat{p}*(1-\hat{p})}{n}}$</font></center><br>
$\hat {p}$: baseline proability for the event to occur

*n*: sample size

- Gross Conversion


```python
G = {}
G['p'] = baseline2['GrCon']
G['n'] = baseline2['clicks'] # the unit of diversion is clicks (after scaled)
G['sd'] = round(math.sqrt(G['p']*(1-G['p'])/G['n']),4)
G['sd']
```




    0.0202



- Retention


```python
R = {}
R['p'] = baseline2['Retention']
R['n'] = baseline2['enrollments'] # the unit of diversion is enrollment (after scaled)
R['sd'] = round(math.sqrt(R['p']*(1-R['p'])/R['n']),4)
R['sd']
```




    0.0549



- Net Conversion


```python
N = {}
N['p'] = baseline2['NetCon']
N['n'] = baseline2['clicks'] # the unit of diversion is clicks (after scaled)
N['sd'] = round(math.sqrt(N['p']*(1-N['p'])/N['n']),4)
N['sd']
```




    0.0156



## 5. Experiment Sizing<a class="anchor" id="5"></a>

### 5.1 Choose sample size<a class="anchor" id="5.1"></a>

In order to calculate the sample size, we need to know the significance level alpha (set to 0.05), statistical power (1-beta, set to 0.8), baseline conversion rate (provided by Udacity) and minimum detectable effect (Dmin, provided by Udacity). We plug these 4 values into this [online calculator](http://www.evanmiller.org/ab-testing/sample-size.html) and get the sample size. Here is an illustration of the sample size online calculator.

![](sample_size.png)

Once we have calulated the sample size, we can convert it to pageviews. Here is a summary of **pageviews** required for each evaluation metrics to achieve targeted statistical power.

Gross Conversion
- Baseline conversion: 20.625%
- Minimum Detectable effect: 1%
- Alpha: 5%
- Power: 80%
- Sample size (calulated using the online calculator): 25,835 enrollments/group
- Number of groups: 2 (experiment and control)
- Total sample size = 25835*2 = 51,670 enrollments
- Clicks/pageviews: 3200/40,000
- Pageview required = enrollments/(clicks/pageviews)=51670/(3200/40000) = 645,875

Retention
- Baseline conversion: 53%
- Minimum Detectable effect: 1%
- Alpha: 5%
- Power: 80%
- Sample size (calulated using the online calculator): 39,155 enrollments/group
- Number of groups: 2 (experiment and control)
- Total sample size = 39155*2 = 78,230 enrollments
- Enrollments/pageviews: 660/40,000
- Pageview required = enrollments/(enrollment/pageviews)=78230/(660/40000) = 4,741,212

Net Conversion
- Baseline conversion: 10.93125%
- Minimum Detectable effect: 0.75%
- Alpha: 5%
- Power: 80%
- Sample size (calulated using the online calculator): 27,413 enrollments/group
- Number of groups: 2 (experiment and control)
- Total sample size = 27413*2 = 54,826 enrollments
- Clicks/pageviews: 3200/40,000
- Pageview required = enrollments/(clicks/pageviews)=54826/(3200/40000) = 685,325

The required pageviews is the maximum of pageviews for all three evaluation metrics, i.e., 4,741,212 pageviews.

### 5.2 Choose duration vs exposure<a class="anchor" id="5.2"></a>

Let's assume we direct 80% of the web traffic to this experiment. Given 40,000 pageviews per day, the experiment would take 148 day to achieve the required 4,741,212 pageviews. This would take too long and is not practical, since We have to wait for 5 months for the experimental results in order to make a business decision. Therefore we have to drop rentention as an evaluation metric.
Now we are left with two evaluation metrics; this reduced the number of pageviews required to 685,325. The experiment will takes about 22 days to achieve the required pageviews.

## 6. Experimental Analysis<a class="anchor" id="6"></a>

### 6.1 Sanity checks<a class="anchor" id="6.1"></a>

Let's check if the **invariant** metrics are randomly split between the control and experiment group. We can assume a binomial distribution here; confidence level is set at 95%. If sanity check fails, look at the day-by-day data to see if they are any anomalies that can give us some insights into what might be the problem. Do not proceed to the rest of the analysis unless all sanity checks pass.

The data are provided by Udacity as two spreadsheets (control and experimental groups). We'll load them into pandas dataframe.


```python
con = pd.read_csv('control.csv')
exp = pd.read_csv('experiment.csv')
print(con)
```

               Date  Pageviews  Clicks  Enrollments  Payments
    0   Sat, Oct 11       7723     687        134.0      70.0
    1   Sun, Oct 12       9102     779        147.0      70.0
    2   Mon, Oct 13      10511     909        167.0      95.0
    3   Tue, Oct 14       9871     836        156.0     105.0
    4   Wed, Oct 15      10014     837        163.0      64.0
    5   Thu, Oct 16       9670     823        138.0      82.0
    6   Fri, Oct 17       9008     748        146.0      76.0
    7   Sat, Oct 18       7434     632        110.0      70.0
    8   Sun, Oct 19       8459     691        131.0      60.0
    9   Mon, Oct 20      10667     861        165.0      97.0
    10  Tue, Oct 21      10660     867        196.0     105.0
    11  Wed, Oct 22       9947     838        162.0      92.0
    12  Thu, Oct 23       8324     665        127.0      56.0
    13  Fri, Oct 24       9434     673        220.0     122.0
    14  Sat, Oct 25       8687     691        176.0     128.0
    15  Sun, Oct 26       8896     708        161.0     104.0
    16  Mon, Oct 27       9535     759        233.0     124.0
    17  Tue, Oct 28       9363     736        154.0      91.0
    18  Wed, Oct 29       9327     739        196.0      86.0
    19  Thu, Oct 30       9345     734        167.0      75.0
    20  Fri, Oct 31       8890     706        174.0     101.0
    21   Sat, Nov 1       8460     681        156.0      93.0
    22   Sun, Nov 2       8836     693        206.0      67.0
    23   Mon, Nov 3       9437     788          NaN       NaN
    24   Tue, Nov 4       9420     781          NaN       NaN
    25   Wed, Nov 5       9570     805          NaN       NaN
    26   Thu, Nov 6       9921     830          NaN       NaN
    27   Fri, Nov 7       9424     781          NaN       NaN
    28   Sat, Nov 8       9010     756          NaN       NaN
    29   Sun, Nov 9       9656     825          NaN       NaN
    30  Mon, Nov 10      10419     874          NaN       NaN
    31  Tue, Nov 11       9880     830          NaN       NaN
    32  Wed, Nov 12      10134     801          NaN       NaN
    33  Thu, Nov 13       9717     814          NaN       NaN
    34  Fri, Nov 14       9192     735          NaN       NaN
    35  Sat, Nov 15       8630     743          NaN       NaN
    36  Sun, Nov 16       8970     722          NaN       NaN
    

The meaning of each column is:
- Pageviews: Number of unique cookies to view the course overview page that day.
- Clicks: Number of unique cookies to click the course overview page that day.
- Enrollments: Number of user-ids to enroll in the free trial that day.
- Payments: Number of user-ids who who enrolled on that day to remain enrolled for 14 days and thus make a payment. (Note that the date for this column is the start date, that is, the date of enrollment, rather than the date of the payment. The payment happened 14 days later. Because of this, the enrollments and payments are tracked for 14 fewer days than the other columns.)

Our three invariant metrics are cookies (pageviews), clicks and CTP (clicks/pageviews). Let's do sanity checks on them one by one.

- Pageview: number of unique cookies to view the course overview page


```python
cookies_con = con['Pageviews'].sum()
cookies_exp = exp['Pageviews'].sum()
cookies_total = cookies_con + cookies_exp
print('number of unique cookies to view the course page in the control group:', cookies_con)
print('number of unique cookies to view the course page in the experiment group:', cookies_exp)
```

    number of unique cookies to view the course page in the control group: 345543
    number of unique cookies to view the course page in the experiment group: 344660
    

We eyeballed the numbers and they look close to each other. We expect the amount of pageviews in the control and experiment groups to be split even, roughtly 50% each. The expected probability of a sample getting assigned to the control group is 0.5. Next we alculate the standard deviation and confidence interval and check if the observed probability p_hat is within the confidence interval.


```python
p = 0.5
p_hat = cookies_con/cookies_total
sd = math.sqrt(p_hat*(1-p_hat)/cookies_total)
m = 1.96*sd
lower_bound, upper_bound = round(p+m,4), round(p-m,4)
print('Obersved p_hat', round(p_hat,4))
print('The confidence interval is ({},{})'.format(lower_bound,upper_bound))
```

    Obersved p_hat 0.5006
    The confidence interval is (0.5012,0.4988)
    

p_hat is within the confidence interval, which means the control and experiments are randomly split. The invariant metric cookies passed the sanity check.

- Clicks: Number of unique cookies to click the course overview page that day.


```python
clicks_con = con['Clicks'].sum()
clicks_exp = exp['Clicks'].sum()
clicks_total = clicks_con + clicks_exp
print('number of unique cookies to click the course page in the control group:', clicks_con)
print('number of unique cookies to click the course page in the experiment group:', clicks_exp)
```

    number of unique cookies to click the course page in the control group: 28378
    number of unique cookies to click the course page in the experiment group: 28325
    


```python
p_hat = clicks_con/clicks_total
sd = math.sqrt(p_hat*(1-p_hat)/clicks_total)
m = 1.96*sd
lower_bound, upper_bound = round(p+m,4), round(p-m,4)
print('Observed p_hat', round(p_hat,4))
print('The confidence interval is ({},{})'.format(lower_bound,upper_bound))
```

    Observed p_hat 0.5005
    The confidence interval is (0.5041,0.4959)
    

- CTP (clicks/cookies)


```python
ctp_con = clicks_con/cookies_con
ctp_exp = clicks_exp/cookies_exp
d_hat = ctp_exp - ctp_con
p_pooled = clicks_total/cookies_total
sd_pooled = math.sqrt(p_pooled*(1-p_pooled)*(1/cookies_con+1/cookies_exp))
m = 1.96*sd_pooled
lower_bound, upper_bound = round(0+m,4), round(0-m,4)
print('Oberved difference', round(d_hat,4))
print('The confidence interval is ({},{})'.format(lower_bound,upper_bound))
```

    Oberved difference 0.0001
    The confidence interval is (0.0013,-0.0013)
    

We can see they all three invariant metrics passed sanity checks.

### 6.2 Effec size tests<a class="anchor" id="6.2"></a>

For **evaluation** metrics, give a 95% confidence level for the difference between the experiment and control groups, calculate its confidence interval. Check if each metric is statistically siginificant and practically significant. 
A metric is statistically significant if the confidence interval does not include 0 (i.e., we can be confident there was a change); and it is practically significant if the confidence interval does not include the practical significance boundary Dmin (i.e., you can be confident there is a change that matters to the business.)

- Gross conversion

Note: the spreadsheet provided by Udacity contains data for cookies and clicks for 37 days, and enrollments and payments for 23 days. So we we work with gross conversion and net conversion, we will only use the data for 23 days. We expect the gross conversion drops in the experimental group.


```python
# Count the total clicks for the 23 days where the 'Enrollments' column is not NaN
clicks_con = con['Clicks'].loc[con['Enrollments'].notnull()].sum()
clicks_exp = exp['Clicks'].loc[exp['Enrollments'].notnull()].sum()
```


```python
enrollments_con = con['Enrollments'].sum()
enrollments_exp = exp['Enrollments'].sum()
g_con = enrollments_con/clicks_con
g_exp = enrollments_exp/clicks_exp
g_diff = g_exp-g_con
g_pooled = (enrollments_con+enrollments_exp)/(clicks_con+clicks_exp)
g_sd = math.sqrt(g_pooled*(1-g_pooled)*(1/clicks_con+1/clicks_exp))
m = 1.96*g_sd
lower_bound, upper_bound = round(g_diff-m,4), round(g_diff+m,4)
print('The change due to the experiment is {}'.format(round(g_diff,4)))
print('The confidence interval is ({},{})'.format(lower_bound,upper_bound))
```

    The change due to the experiment is -0.0206
    The confidence interval is (-0.0291,-0.012)
    

The observed change is statistically significant since the confidence interval does not include 0. And it is practically significant since the confidence interval does not include the minimum detectable effect -0.01.

The change is both statiscally and practically signifcant. We have a negative change of 2.06%, which means the gross conversion rate in the experiment group has dropped 2.06% and this change is significant. This matches with our expecation, as when customers click the "Start free trial" button, they are asked how many hours they can dedicate to the study per week; this change should make customers think twice before they go ahead and enroll, hence the decrease gross conversion rate.

- Net conversion


```python
payments_con = con['Payments'].sum()
payments_exp = exp['Payments'].sum()
n_con = payments_con/clicks_con
n_exp = payments_exp/clicks_exp
n_diff = n_exp-n_con
n_pooled = (payments_con+payments_exp)/(clicks_con+clicks_exp)
n_sd = math.sqrt(n_pooled*(1-n_pooled)*(1/clicks_con+1/clicks_exp))
m = 1.96*n_sd
lower_bound, upper_bound = round(n_diff-m,4), round(n_diff+m,4)
print('The change due to the experiment is {}'.format(round(n_diff,4)))
print('The confidence interval is ({},{})'.format(lower_bound,upper_bound))
print('The observed change is statistically significant if the confidence interval does not include 0. And it is practically significant if it does not include the minimum detectable effect -0.0075.')
```

    The change due to the experiment is -0.0049
    The confidence interval is (-0.0116,0.0019)
    The observed change is statistically significant if the confidence interval does not include 0. And it is practically significant if it does not include the minimum detectable effect -0.0075.
    

We got a very small change of -0.49%, which is neither statistically significant, nor practically significant.

### 6.3 Sign tests<a class="anchor" id="6.3"></a>

For each **evalution** metric, let's count day-by-day how many days in the experiment group is lower than the control group out of 23 days, and this is the number of successes for our binomial model. We then cauculate the p-value and compare it with alpha.


```python
# Let's merge the two dataframes since we need to compare day-to-day gross conversion and net conversion in control and expeirment group
combine = con.join(exp, how='inner',lsuffix='_con', rsuffix='_exp')
combine
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Date_con</th>
      <th>Pageviews_con</th>
      <th>Clicks_con</th>
      <th>Enrollments_con</th>
      <th>Payments_con</th>
      <th>Date_exp</th>
      <th>Pageviews_exp</th>
      <th>Clicks_exp</th>
      <th>Enrollments_exp</th>
      <th>Payments_exp</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Sat, Oct 11</td>
      <td>7723</td>
      <td>687</td>
      <td>134.0</td>
      <td>70.0</td>
      <td>Sat, Oct 11</td>
      <td>7716</td>
      <td>686</td>
      <td>105.0</td>
      <td>34.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Sun, Oct 12</td>
      <td>9102</td>
      <td>779</td>
      <td>147.0</td>
      <td>70.0</td>
      <td>Sun, Oct 12</td>
      <td>9288</td>
      <td>785</td>
      <td>116.0</td>
      <td>91.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Mon, Oct 13</td>
      <td>10511</td>
      <td>909</td>
      <td>167.0</td>
      <td>95.0</td>
      <td>Mon, Oct 13</td>
      <td>10480</td>
      <td>884</td>
      <td>145.0</td>
      <td>79.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Tue, Oct 14</td>
      <td>9871</td>
      <td>836</td>
      <td>156.0</td>
      <td>105.0</td>
      <td>Tue, Oct 14</td>
      <td>9867</td>
      <td>827</td>
      <td>138.0</td>
      <td>92.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Wed, Oct 15</td>
      <td>10014</td>
      <td>837</td>
      <td>163.0</td>
      <td>64.0</td>
      <td>Wed, Oct 15</td>
      <td>9793</td>
      <td>832</td>
      <td>140.0</td>
      <td>94.0</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Thu, Oct 16</td>
      <td>9670</td>
      <td>823</td>
      <td>138.0</td>
      <td>82.0</td>
      <td>Thu, Oct 16</td>
      <td>9500</td>
      <td>788</td>
      <td>129.0</td>
      <td>61.0</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Fri, Oct 17</td>
      <td>9008</td>
      <td>748</td>
      <td>146.0</td>
      <td>76.0</td>
      <td>Fri, Oct 17</td>
      <td>9088</td>
      <td>780</td>
      <td>127.0</td>
      <td>44.0</td>
    </tr>
    <tr>
      <th>7</th>
      <td>Sat, Oct 18</td>
      <td>7434</td>
      <td>632</td>
      <td>110.0</td>
      <td>70.0</td>
      <td>Sat, Oct 18</td>
      <td>7664</td>
      <td>652</td>
      <td>94.0</td>
      <td>62.0</td>
    </tr>
    <tr>
      <th>8</th>
      <td>Sun, Oct 19</td>
      <td>8459</td>
      <td>691</td>
      <td>131.0</td>
      <td>60.0</td>
      <td>Sun, Oct 19</td>
      <td>8434</td>
      <td>697</td>
      <td>120.0</td>
      <td>77.0</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Mon, Oct 20</td>
      <td>10667</td>
      <td>861</td>
      <td>165.0</td>
      <td>97.0</td>
      <td>Mon, Oct 20</td>
      <td>10496</td>
      <td>860</td>
      <td>153.0</td>
      <td>98.0</td>
    </tr>
    <tr>
      <th>10</th>
      <td>Tue, Oct 21</td>
      <td>10660</td>
      <td>867</td>
      <td>196.0</td>
      <td>105.0</td>
      <td>Tue, Oct 21</td>
      <td>10551</td>
      <td>864</td>
      <td>143.0</td>
      <td>71.0</td>
    </tr>
    <tr>
      <th>11</th>
      <td>Wed, Oct 22</td>
      <td>9947</td>
      <td>838</td>
      <td>162.0</td>
      <td>92.0</td>
      <td>Wed, Oct 22</td>
      <td>9737</td>
      <td>801</td>
      <td>128.0</td>
      <td>70.0</td>
    </tr>
    <tr>
      <th>12</th>
      <td>Thu, Oct 23</td>
      <td>8324</td>
      <td>665</td>
      <td>127.0</td>
      <td>56.0</td>
      <td>Thu, Oct 23</td>
      <td>8176</td>
      <td>642</td>
      <td>122.0</td>
      <td>68.0</td>
    </tr>
    <tr>
      <th>13</th>
      <td>Fri, Oct 24</td>
      <td>9434</td>
      <td>673</td>
      <td>220.0</td>
      <td>122.0</td>
      <td>Fri, Oct 24</td>
      <td>9402</td>
      <td>697</td>
      <td>194.0</td>
      <td>94.0</td>
    </tr>
    <tr>
      <th>14</th>
      <td>Sat, Oct 25</td>
      <td>8687</td>
      <td>691</td>
      <td>176.0</td>
      <td>128.0</td>
      <td>Sat, Oct 25</td>
      <td>8669</td>
      <td>669</td>
      <td>127.0</td>
      <td>81.0</td>
    </tr>
    <tr>
      <th>15</th>
      <td>Sun, Oct 26</td>
      <td>8896</td>
      <td>708</td>
      <td>161.0</td>
      <td>104.0</td>
      <td>Sun, Oct 26</td>
      <td>8881</td>
      <td>693</td>
      <td>153.0</td>
      <td>101.0</td>
    </tr>
    <tr>
      <th>16</th>
      <td>Mon, Oct 27</td>
      <td>9535</td>
      <td>759</td>
      <td>233.0</td>
      <td>124.0</td>
      <td>Mon, Oct 27</td>
      <td>9655</td>
      <td>771</td>
      <td>213.0</td>
      <td>119.0</td>
    </tr>
    <tr>
      <th>17</th>
      <td>Tue, Oct 28</td>
      <td>9363</td>
      <td>736</td>
      <td>154.0</td>
      <td>91.0</td>
      <td>Tue, Oct 28</td>
      <td>9396</td>
      <td>736</td>
      <td>162.0</td>
      <td>120.0</td>
    </tr>
    <tr>
      <th>18</th>
      <td>Wed, Oct 29</td>
      <td>9327</td>
      <td>739</td>
      <td>196.0</td>
      <td>86.0</td>
      <td>Wed, Oct 29</td>
      <td>9262</td>
      <td>727</td>
      <td>201.0</td>
      <td>96.0</td>
    </tr>
    <tr>
      <th>19</th>
      <td>Thu, Oct 30</td>
      <td>9345</td>
      <td>734</td>
      <td>167.0</td>
      <td>75.0</td>
      <td>Thu, Oct 30</td>
      <td>9308</td>
      <td>728</td>
      <td>207.0</td>
      <td>67.0</td>
    </tr>
    <tr>
      <th>20</th>
      <td>Fri, Oct 31</td>
      <td>8890</td>
      <td>706</td>
      <td>174.0</td>
      <td>101.0</td>
      <td>Fri, Oct 31</td>
      <td>8715</td>
      <td>722</td>
      <td>182.0</td>
      <td>123.0</td>
    </tr>
    <tr>
      <th>21</th>
      <td>Sat, Nov 1</td>
      <td>8460</td>
      <td>681</td>
      <td>156.0</td>
      <td>93.0</td>
      <td>Sat, Nov 1</td>
      <td>8448</td>
      <td>695</td>
      <td>142.0</td>
      <td>100.0</td>
    </tr>
    <tr>
      <th>22</th>
      <td>Sun, Nov 2</td>
      <td>8836</td>
      <td>693</td>
      <td>206.0</td>
      <td>67.0</td>
      <td>Sun, Nov 2</td>
      <td>8836</td>
      <td>724</td>
      <td>182.0</td>
      <td>103.0</td>
    </tr>
    <tr>
      <th>23</th>
      <td>Mon, Nov 3</td>
      <td>9437</td>
      <td>788</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>Mon, Nov 3</td>
      <td>9359</td>
      <td>789</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>24</th>
      <td>Tue, Nov 4</td>
      <td>9420</td>
      <td>781</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>Tue, Nov 4</td>
      <td>9427</td>
      <td>743</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>25</th>
      <td>Wed, Nov 5</td>
      <td>9570</td>
      <td>805</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>Wed, Nov 5</td>
      <td>9633</td>
      <td>808</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>26</th>
      <td>Thu, Nov 6</td>
      <td>9921</td>
      <td>830</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>Thu, Nov 6</td>
      <td>9842</td>
      <td>831</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>27</th>
      <td>Fri, Nov 7</td>
      <td>9424</td>
      <td>781</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>Fri, Nov 7</td>
      <td>9272</td>
      <td>767</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>28</th>
      <td>Sat, Nov 8</td>
      <td>9010</td>
      <td>756</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>Sat, Nov 8</td>
      <td>8969</td>
      <td>760</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>29</th>
      <td>Sun, Nov 9</td>
      <td>9656</td>
      <td>825</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>Sun, Nov 9</td>
      <td>9697</td>
      <td>850</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>30</th>
      <td>Mon, Nov 10</td>
      <td>10419</td>
      <td>874</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>Mon, Nov 10</td>
      <td>10445</td>
      <td>851</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>31</th>
      <td>Tue, Nov 11</td>
      <td>9880</td>
      <td>830</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>Tue, Nov 11</td>
      <td>9931</td>
      <td>831</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>32</th>
      <td>Wed, Nov 12</td>
      <td>10134</td>
      <td>801</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>Wed, Nov 12</td>
      <td>10042</td>
      <td>802</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>33</th>
      <td>Thu, Nov 13</td>
      <td>9717</td>
      <td>814</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>Thu, Nov 13</td>
      <td>9721</td>
      <td>829</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>34</th>
      <td>Fri, Nov 14</td>
      <td>9192</td>
      <td>735</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>Fri, Nov 14</td>
      <td>9304</td>
      <td>770</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>35</th>
      <td>Sat, Nov 15</td>
      <td>8630</td>
      <td>743</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>Sat, Nov 15</td>
      <td>8668</td>
      <td>724</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>36</th>
      <td>Sun, Nov 16</td>
      <td>8970</td>
      <td>722</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>Sun, Nov 16</td>
      <td>8988</td>
      <td>710</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>




```python
combine.count()
```




    Date_con           37
    Pageviews_con      37
    Clicks_con         37
    Enrollments_con    23
    Payments_con       23
    Date_exp           37
    Pageviews_exp      37
    Clicks_exp         37
    Enrollments_exp    23
    Payments_exp       23
    dtype: int64




```python
# We only need rows where Enrollments and Payments are not NaN
combine = combine.loc[combine['Enrollments_con'].notnull()]
combine.count()
```




    Date_con           23
    Pageviews_con      23
    Clicks_con         23
    Enrollments_con    23
    Payments_con       23
    Date_exp           23
    Pageviews_exp      23
    Clicks_exp         23
    Enrollments_exp    23
    Payments_exp       23
    dtype: int64




```python
# We calcuate gross conversion and net conversion, then compare the day-to-day data from control and experiment group.
# If the experiment value is larger than the control group, the sign is 1, otherwise 0.
combine.loc[:,'g_con'] = combine['Enrollments_con']/combine['Clicks_con']
combine.loc[:,'g_exp'] = combine['Enrollments_exp']/combine['Clicks_exp']
combine.loc[:,'g_sign'] = np.where(combine['g_exp']>combine['g_con'],1,0)
combine.loc[:,'n_con'] = combine['Payments_con']/combine['Clicks_con']
combine.loc[:,'n_exp'] = combine['Payments_exp']/combine['Clicks_exp']
combine.loc[:,'n_sign'] = np.where(combine['n_exp']>combine['n_con'],1,0)
combine[['g_sign','n_sign']]
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>g_sign</th>
      <th>n_sign</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>0</td>
      <td>1</td>
    </tr>
    <tr>
      <th>2</th>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>0</td>
      <td>1</td>
    </tr>
    <tr>
      <th>5</th>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>6</th>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>7</th>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>8</th>
      <td>0</td>
      <td>1</td>
    </tr>
    <tr>
      <th>9</th>
      <td>0</td>
      <td>1</td>
    </tr>
    <tr>
      <th>10</th>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>11</th>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>12</th>
      <td>0</td>
      <td>1</td>
    </tr>
    <tr>
      <th>13</th>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>14</th>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>15</th>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>16</th>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>17</th>
      <td>1</td>
      <td>1</td>
    </tr>
    <tr>
      <th>18</th>
      <td>1</td>
      <td>1</td>
    </tr>
    <tr>
      <th>19</th>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <th>20</th>
      <td>1</td>
      <td>1</td>
    </tr>
    <tr>
      <th>21</th>
      <td>0</td>
      <td>1</td>
    </tr>
    <tr>
      <th>22</th>
      <td>0</td>
      <td>1</td>
    </tr>
  </tbody>
</table>
</div>




```python
g_sum = sum(combine['g_sign'])
n_sum = sum(combine['n_sign'])
total = combine.shape[0]
print('Number of Days where gross conversion in the experiment is larger than that in the control group:',g_sum)
print('Number of Days where net conversion in the experiment is larger than that in the control group:',n_sum)
print('Total number of days:',total)
```

    Number of Days where gross conversion in the experiment is larger than that in the control group: 4
    Number of Days where net conversion in the experiment is larger than that in the control group: 10
    Total number of days: 23
    

Now let's calculate the p-value using the binomial distribution equation.
<br>
<center><font size="4"> $p(successes )=\frac{n!}{x!(n-x)!}p^x(1-p)^{n-x}$ </font></center>

After we count the number of days in which the experiment has a higher metric value than that of the control group (we define this event is a "success"), we need to decide if the p-value is statistically significant (< 0.05). On any day, if the event is a "success" or not is random, therefore _p_ = 0.5. n is total number of days, which is 23. x is the number of days being a "success" (the value in the experiment group is larger than that in the control group).

$p-value$ is the probability of observing an event equal to or more extreme than that observed. If we observed 4 successes, the $p-value$ for the test is:
<br>
<center>$P(x<=4)=P(0)+P(1)+P(2)+P(3)+P(4)$.</center><br>

Because this is a two-tailed test, $p-value$ will be doubled and compared to 0.05.


```python
from scipy.stats import binom
n = 23
p = 0.5
# define a function that calculate the two-tailed p-value for a given x
def twotail_pvalue(x,n,p):
    p_x = sum(binom.pmf(i,n,p) for i in range(x+1))
    return round(2*p_x,4)
```


```python
g_pvalue = twotail_pvalue(g_sum,n,p)
n_pvalue = twotail_pvalue(n_sum,n,p)
print(g_pvalue)
print(n_pvalue)
```

    0.0026
    0.6776
    

|Evaluation metrics|p-value for the sign test|Statiscally significant at alpha=0.05|
|:----------------:|:-----------------------:|:-----------------------------------:|
|Gross conversion|0.0026|Yes|
|Net conversion|0.6776|No|

The sensitivity of a sign test is usually lower than that of a effect size test. Still, we get the same conclusion from the sign test as we get from the effect size test above. The change in gross conversion is significant, while the change in net conversion is not.

## 7. Conclusion and Recommendation<a class="anchor" id="7"></a>

We wanted to determine if making a change after a student clicks the "start free trial" button can have a significant impact on gross conversion rate and net conversion. Therefor, we designed an A/B Testing experiment. Udacity students (tracked by cookies) were directed randomly into two groups, control and experiment. After clicking the "start free trial" button, the experiment group was asked how many hours per week they can devote to learning, while the control group was not.

Three invariant metrics (number of cookies, number of clicks and Click-Through-Probability) were chosen and have passed sanity checks. Gross conversion (enrollments/cookies) and net conversion (payments/cookies) were chosen as evaluation metrics. Practical significance threasholds (Dmin) were set for each evaluation metrics.

The null hypothesis H<sub>0</sub> that there is no significant difference in the evaluation metrics between the two groups. To reject the null hypothesis, the difference between the groups should be statitically significant, as well as exceed Dmin, for **all** evaluation metrics.

The experiment results showed that gross conversion was found to be statisticcaly significant with alpha = 0.05, and exceeded the practical significance threahold. Net conversion, on the other hand, is neither statistically significant nor practically significant.

The purpose of the A/B testin experiment was to determine if by adding a step qualifying if a student can dedicate enough study time, we can improve the overall student experimence, thus reducing the number of frustrated students who left the free trial because they didn't have enough time; while at the same time without significantly reducing the number of students to continue past the free trial. A statistically and practically significant drop in gross conversion was observed; however, no significant change in net conversion was observed. This means a descrease in enrollment, and at the same time no increase in students staying past the 14 day free trial leading to payment. Based on the results, we do not recommend lauching the change at the moment without conducting follow-up experiments.

## 8. Follow-up Experiment<a class="anchor" id="8"></a>

Retention (payments/enrollments) was initially chosen as an evaluation metrics but we didn't run the experiment using this metric or calculate its statistical significance, because it would take about 5 months to achieve the required pageviews. In reality where data are different and it might not need 5 months to achieve the required sample size, a company might decide retention is an important metric we are interested in and we want to keep track of it. If a statistically significant and practically significant retention change is observed, it means we are seeing more students are staying and paying after the 14 day trial period ends, out of all the students who are enrolled, and we would recommend launching the experiment.
