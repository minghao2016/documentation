Last summer we discussed the simplified interface of the [1.0 CRAN
release of
healthcare.ai-R](https://healthcare.ai/healthcareai-r-1-0-features-simplier-interface/),
and we’re now thrilled to demo new features related to clinician
guidance in the [1.2
version](https://cran.r-project.org/web/packages/healthcareai/healthcareai.pdf).
We’re calling this Patient Impact Predictor (PIP).

### Patients like this, should be treated like this

This week we’d like to highlight new functionality that allows one to go
a step beyond surfacing predictions to also surface targeted
interventions. Risk scores are a great first step, but [prescriptive
guidance](https://en.wikipedia.org/wiki/Prescriptive_analytics#Applications_in_healthcare)
is where results ML may actually catch up to the hype. For example, it’s
very useful to know that Eddy Exampleton has a 57% risk of heart
failure. In past versions of healthcare.ai, we offered not only that but
also a list top three variables responsible for that patient’s high risk
score.

But what if

-   those varialbes are unmodifiable (like age, race, etc)?
-   the clinician wants data-driven guidance on appropriate action to
    take?

For example, we might want to know how Eddy’s risk changes if a certain
medical procedure is performed, if a different medication is prescribed,
or if Eddy works to change his blood pressure. As with most machine
learning in healthcare, to get the most of out this functionality, one
should leverage subject-matter expertise: in our case, subject-matter
expertise is critical for carefully selecting the variables on which to
make recommendations.

How do we do this? As is often the case with healthcare.ai, we take try
to incorporate the most practical techniques from machine learning and
apply it to healthcare decision support. En route to this prescriptive
breakthrough, we were inspired by
[LIME](https://arxiv.org/pdf/1602.04938v1.pdf), which is used for [model
interpretability](https://christophm.github.io/interpretable-ml-book/lime.html).

### An example

Let’s illustrate how this works using the UCI Machine Learning
Repository’s [Fertility
Dataset](https://archive.ics.uci.edu/ml/datasets/Fertility), a 100 row
data set capturing information about how various factors are related to
male fertility. After doing some slight feature engineering, here’s a
sample of the pertinent varialbes:

-   `id`: an ID column to use as the grain: this is just the row number
    in the dataset.
-   `age`: the age of the patient
-   `trauma`: whether or not the patient was in an accident or
    experienced serious trauma
-   `intervention`: whether or not a surgical intervention was peformed
-   `alcohol`: alcohol consumption habits, grouped into just three
    categories (`several_a_week`, `once_a_week`, and `hardly_ever`)
-   `smoking`: smoking habits (`daily`, `occasional`, `never`)
-   `hours_sitting`: the average daily number of hours spent sitting
-   `altered_fertility`: whether or not the patient has altered
    fertility

You can download our ML-ready version of this simple dataset [here]().
After choosing `infertility` as our label or predicted column, we train
and deploy a random forest model on separate parts of the data (see
[here]() for the code). Now, after generating these predictions, we can
use the `getProcessVariablesDf` function to generate recommendations.
The simplest way to use this function is to pass a vector of names of
categorical variables for which we desire to make recommendations. Some
of the variables are outside of our control (the person’s age, whether
they experienced serious trauma, etc.), but other variables are more
ammenable to recommendations. For example, to see how smoking habits,
alcohol consumption, and surgical interventions might affect fertility,
we would run the following command:

    pvdf1 <- rfD$getProcessVariablesDf(modifiableVariables = c("smoking", "intervention", "alcohol"))

This returns a dataframe (i.e., a table) of prescriptive guidanance–we
present the top recommended intervention for three patients below. See
if you can spot the potential issue that arises from letting ML loose
without [SME](https://en.wikipedia.org/wiki/Subject-matter_expert)
input.

<table>
<thead>
<tr class="header">
<th style="text-align: right;">id</th>
<th style="text-align: right;">PredictedProbNBR</th>
<th style="text-align: left;">Modify1TXT</th>
<th style="text-align: left;">Modify1Current</th>
<th style="text-align: left;">Modify1AltValue</th>
<th style="text-align: right;">Modify1AltPrediction</th>
<th style="text-align: right;">Modify1Delta</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td style="text-align: right;">5</td>
<td style="text-align: right;">0.59</td>
<td style="text-align: left;">alcohol</td>
<td style="text-align: left;">once_a_week</td>
<td style="text-align: left;">hardly_ever</td>
<td style="text-align: right;">0.25</td>
<td style="text-align: right;">-0.35</td>
</tr>
<tr class="even">
<td style="text-align: right;">6</td>
<td style="text-align: right;">0.16</td>
<td style="text-align: left;">alcohol</td>
<td style="text-align: left;">once_a_week</td>
<td style="text-align: left;">hardly_ever</td>
<td style="text-align: right;">0.12</td>
<td style="text-align: right;">-0.04</td>
</tr>
<tr class="odd">
<td style="text-align: right;">7</td>
<td style="text-align: right;">0.18</td>
<td style="text-align: left;">smoking</td>
<td style="text-align: left;">never</td>
<td style="text-align: left;">occasional</td>
<td style="text-align: right;">0.18</td>
<td style="text-align: right;">-0.01</td>
</tr>
</tbody>
</table>

Going left to right, we have the patient identifier, the predicted
probability for the original row of data, and then the top
recommendation for each patient. `Modify1TXT` gives the name of the
variable on which we’re making a recommendation, followed by the current
value (`Modify1Current`) and the alternate baseline (`Modify1AltValue`)
for that variable. The `Modify1AltPrediction` provides the patient’s
risk if that recommendation is acted upon and the `Modify1Delta` shows
how much of a risk reduction that action could provide for *that*
patient.

But, did you catch the error?! The guidance for patient 5 and 6 is
reasonable–lowering your alcohol consumption is a [clinically
sensible](http://www.bmj.com/content/317/7157/505?linkType=FULL&resid=317/7157/505&journalCode=bmj)
route to increased fertility. For patient 7, however, why would this ML
guidance suggest they start smoking? Broadly, that’s what happens when
you don’t pair ML with subject matter expertise. Occasionally there are
local patterns in the model that don’t translate to reality. For
example, due to the [curse of
high-dimensionality](https://en.wikipedia.org/wiki/Curse_of_dimensionality)
you’ll occasionally find that in local regions without much data
recommendations arise to smoke or to increase one’s alcohol consumption.

To mitigate this, the user of healthcare.ai (hopefully with guidance
from a [SME](https://en.wikipedia.org/wiki/Subject-matter_expert))
inputs the baseline that makes sense for a given modifiable risk factor.
This avoids falling into regions where the true signal is obscured. Here
we simply add guidance for the `smoking` and `alcohol` risk features for
the healthy baseline to be `never` and `hardly_ever`, respectively.

    pvdf2 <- rfD$getProcessVariablesDf(modifiableVariables = c("smoking",
                                                               "intervention",
                                                               "alcohol"),
                                       variableLevels = list(smoking = c("never"),
                                                             alcohol = c("hardly_ever")))

And here is the first set of recommendations for patients 5 through 7:

<table>
<thead>
<tr class="header">
<th style="text-align: right;">id</th>
<th style="text-align: right;">PredictedProbNBR</th>
<th style="text-align: left;">Modify1TXT</th>
<th style="text-align: left;">Modify1Current</th>
<th style="text-align: left;">Modify1AltValue</th>
<th style="text-align: right;">Modify1AltPrediction</th>
<th style="text-align: right;">Modify1Delta</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td style="text-align: right;">5</td>
<td style="text-align: right;">0.59</td>
<td style="text-align: left;">alcohol</td>
<td style="text-align: left;">once_a_week</td>
<td style="text-align: left;">hardly_ever</td>
<td style="text-align: right;">0.25</td>
<td style="text-align: right;">-0.35</td>
</tr>
<tr class="even">
<td style="text-align: right;">6</td>
<td style="text-align: right;">0.16</td>
<td style="text-align: left;">alcohol</td>
<td style="text-align: left;">once_a_week</td>
<td style="text-align: left;">hardly_ever</td>
<td style="text-align: right;">0.12</td>
<td style="text-align: right;">-0.04</td>
</tr>
<tr class="odd">
<td style="text-align: right;">7</td>
<td style="text-align: right;">0.18</td>
<td style="text-align: left;">smoking</td>
<td style="text-align: left;">never</td>
<td style="text-align: left;">never</td>
<td style="text-align: right;">0.18</td>
<td style="text-align: right;">0.00</td>
</tr>
</tbody>
</table>

Note that now the ML advice is restricted to what makes sense clinically
and intervention priorities are established in a data-driven and
actionable way.

For example, the PIP recommendations are customized to each patient: the
top recommendation for patient 5 is different from the top
recommendation for patient 6. Looking at the delta column (on the far
right), we see that some recommendations are more effective than others.
Patient risk impactability isn’t the same for everyone. For example,
modifying alcohol consumption for patient 5 leads to a noticable drop in
the risk, while the effect is much smaller if patient 6 lowered their
alcohol consumption.

Finally, the results for patient 7 may seem odd at first glance: the top
recommendation is not to change their smoking habits. Looking closer, we
see that this patient already has good habits, so it makes sense that
there may not be anything actionable to lower their risk of infertility.
Note that having fewer modifiable variables and fewer rows in your
dataset will make it such that the model has a more difficult time
producing recommendations for folks that are either healthy or not
impactable.

Let’s look at the second set of recommendations for these same patients:

<table>
<thead>
<tr class="header">
<th style="text-align: right;">id</th>
<th style="text-align: right;">PredictedProbNBR</th>
<th style="text-align: left;">Modify2TXT</th>
<th style="text-align: left;">Modify2Current</th>
<th style="text-align: left;">Modify2AltValue</th>
<th style="text-align: right;">Modify2AltPrediction</th>
<th style="text-align: right;">Modify2Delta</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td style="text-align: right;">5</td>
<td style="text-align: right;">0.59</td>
<td style="text-align: left;">intervention</td>
<td style="text-align: left;">Y</td>
<td style="text-align: left;">N</td>
<td style="text-align: right;">0.32</td>
<td style="text-align: right;">-0.28</td>
</tr>
<tr class="even">
<td style="text-align: right;">6</td>
<td style="text-align: right;">0.16</td>
<td style="text-align: left;">intervention</td>
<td style="text-align: left;">N</td>
<td style="text-align: left;">N</td>
<td style="text-align: right;">0.16</td>
<td style="text-align: right;">0.00</td>
</tr>
<tr class="odd">
<td style="text-align: right;">7</td>
<td style="text-align: right;">0.18</td>
<td style="text-align: left;">intervention</td>
<td style="text-align: left;">Y</td>
<td style="text-align: left;">Y</td>
<td style="text-align: right;">0.18</td>
<td style="text-align: right;">0.00</td>
</tr>
</tbody>
</table>

Note that some folks (like patient 5) are shown to be quite impactable,
in that there are two risk factors changes that could lead to
significant reductions in risk! 1) get them to stop drinking and 2)
cancel that planned surgery (if clinicians weren’t convinced of its
necessity).

By default, this PIP-based suggestive guidance contains the top three
interventions for each patient, though for brevity we’ve only shown two.

### Focusing on impactable patients

Note that one of the main benefits of this new functionality is that the
nurse manager or bed-side clinician is able to focus resources on the
most *impactable* patients. This differs from most healthcare decision
support and early versions of healthcare.ai, where the common framework
is to stratify patients based on risk, which is a less actionable way of
managing your cohort. Focusing on impactability is best and can be done
now by simply sorting the patient list by the `Modify1Delta` column
above, either when discussing a csv file or surfacing this guidance in a
visualization.

### Motivation: machine learning model interpretability

Machine learning models are very good at learning from historical data
to make accurate predictions on new data. In order to make such accurate
predictions, we often must [sacrifice
interpretability](https://healthcare.ai/machine-learning-versus-statistics-use/).
In many fields such as healthcare, we would like to make use of the
strong predictive power of machine learning, but we can [run into
trouble](http://people.dbmi.columbia.edu/noemie/papers/15kdd.pdf) if the
model is too opaque. This problem has led to renewed focus on machine
learning interpretability and tools such as
[LIME](https://homes.cs.washington.edu/~marcotcr/blog/lime/),
[FairML](http://blog.fastforwardlabs.com/2017/03/09/fairml-auditing-black-box-predictive-models.html),
and [many
others](https://www.oreilly.com/ideas/ideas-on-interpreting-machine-learning)

Some model interpretability is already included in the [healthcare.ai
package](http://healthcareai-r.readthedocs.io/en/latest/): Random forest
provides an ordered list of important variables and we can fully
directly understand the Lasso’s predictions by working with the model
coefficients. We also provided some row-level model interpretability in
the top factors that accompany the [predicted risk
scores](http://healthcareai-r.readthedocs.io/en/latest/comparing-and-deploying/deploy).

Understanding how a model works is useful for determining how to act on
a prediction (if we know that high blood pressure contributed strongly
to a high risk prediction, we can take steps to reduce blood pressure).
But we wanted to take this a step further and directly surface
recommendations on a *per-patient* basis.

### The basic idea

The idea behind the recommendations is fairly simple. Recommendations
are made using counterfactual predictions: that is, we take the true
attributes of a patient, changes values in a controlled way, and use the
model to make new predictions for the modified data.

For example, patient 39 is 30 years old, did not have a surgical
intervention, smokes occasionaly, drinks several alcoholic beverages
each week, and spends 8.5 hours sitting per day (on average). The model
predicts that this patient has a 27.5% chance of altered fertility. Now,
we copy this patient’s data, modifying only their smoking behavior to
get a patient who is 30 years old, did not have a surgical intervention,
*doesn’t smoke*, drinks several alcoholic beverages each week, and
spends 8.5 hours sitting per day. Feeding this fictional modified data
to the model we get back a 26.6% risk, down 0.9%. Next, we copy the
original patient’s data again, this time modifying the alcohol
consumption to get a patient who is 30 years old, did not have a
surgical intervention, smokes occasionally, *rarely drinks alcohol*, and
spends 8.5 hours sitting per day. Now the model returns a probability of
15.9%, down 12.6% from the original prediction.

We repeat this process for several different values of several different
variables (for example, we might want to check the effect of reducing
alcohol consumption to just one beverage per week). At the end of this
process, we find that greatly reducing alcohol consumption led to the
biggest predicted reduction in risk (from 27.5% down to 15.9%), followed
by the presence of a surgical intervention (from 27.5% down to 23.1%),
and then smoking cessation (from 27.5% down to 26.6%).

### Flexibility and pitfalls

The function `getProcessVariablesDf` takes some optional parameters that
can be set. We describe some of these in this section. More details and
examples can be found in the documentation via either
`?RandomForestDeployment` or `?LassoDeployment`.

#### Customizing the guidance

You may have noticed that there were no positive delta values in the
example above. Because the positive (yes) class usually corresponds to
an undesirable outcome in healthcare (readmission, infection, etc.), the
default behavior is to surface recommendations which reduce the
predicted probability. This can be reversed in cases where the positive
class represents a desirable outcome. It’s also possible to include as
many recommendations as you’d like or to restrict to fewer than 3.
Finally, by default we surface only one recommendation for each
variable, but this can also be toggled: for example, the top
recommendation might be to heavily reduce alcohol consumption but we
might still want to know the effects of a smaller reduction.

#### Continuous variables and baselines

In the fertility example, all of our modifiable variables were
categorical variables. To get recommendations for continuous variables,
you have to do a little more work. This is because continuous variables
are tricky: In order to make good recommendations, we need some extra
information which can be difficult to automatically extract from the
data. We can get an idea of the complications by studying a model which
predicts which patients are most likely to pay their medical bills. The
patient’s balance seems like a potential variable to use in making
recommendations. For example, it may be worth it to reduce a patient’s
balance if it significantly increases the likelihood that they will pay
their bill. Here are the main difficulties we run into:

1.  Useful recommendations on continuous variables need to vary in
    magnitude: In our fertility model, there were only three values the
    smoking variable could take, but a continuous variable can take many
    more values. For example, in the model to predict likelihood of bill
    payment, a $50 discount might make a difference for a patient who
    owes $200, but is unlikely to affect a patient who owes $100,000.
2.  Often, there are important restrictions we need to impose on our
    recommendations. We probably don’t want to reduce a patient’s
    balance all the way down to $0. Even worse, we don’t want to
    increase the patient’s balance just because the model thinks that
    would improve their chance of payment. Anomalies like this are
    especially a concern with smaller datasets where real-world
    relationships aren’t always well reflected in the data.

To address these issues, we allow for recommendations on continuous
variables, but only if comparison baselines are explicitly specified by
the user. For example, in our fertility model we can make
recommendations about the variable `hours_sitting`, but have to specify
levels. To use a different illustration, if the initial health benefits
of LDL reduction can largely be seen by getting down to 140 mg/dL, with
a subsequent boost by getting down to 120 mg/dL (*for example*), this
type of multiple baseline setup can be handled by this healthcare.ai PIP
framework.

#### Which variables to use

Selecting the right variables to use as modifiable variables can greatly
affect the usefulness of the recommendations. Here are a few issues to
keep in mind:

-   Beware of correlation: The counterfactual predictions involve only
    modifying one variable at a time. If you have several predictor
    variables that are strongly correlated then the recommendations may
    not accurately reflect reality.
-   Consider the relationship to other factors not obviously included in
    the data: Machine learning models have an uncanny knack of
    discovering subtly encoded information which is why they make such
    good predictions, but also why they are susceptible to problems such
    as [data
    leakage](https://medium.com/@colin.fraser/the-treachery-of-leakage-56a2d7c4e931).
    If a variable is acting as a
    [surrogate](https://en.wikipedia.org/wiki/Omitted-variable_bias) for
    an unseen variable, then that variable may not be useful for
    recommendations even if it is very useful when making predictions.
-   For best guidance, find more variables over which you have direct
    control: for example, a practioner can control whether or not a
    patient is referred to a smoking cessation program, but cannot
    directly control the patient’s smoking habits.

To deal with these issues, carefully study the data when building the
model, consult with subject-matter experts when selecting which
variables to work with, and then check the recommendations on new data.

As always, we’d love to hear your feedback! Check out the [Slack
group](http://healthcare.ai/slack/) and say hello. Thanks for reading!

*Note: the Patient Impact Predictor project (codename: LIMONE) was
started and driven by the esteemed Yannick Van Huele, while a data
science intern at [Health Catalyst](https://www.healthcatalyst.com/).*
