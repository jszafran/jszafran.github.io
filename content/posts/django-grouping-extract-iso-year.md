---
title: "Django & grouping by year of the week"
date: 2022-08-15T10:09:55+02:00
draft: false
tags: ['django', 'python', 'web development']
categories: ['django']
---

I've been recently working on a task at my job that involved debugging some aggregated calculations in Django (group by year of the week - i.e. 2022 Week 1, 2022 Week 2, 2022 Week 3 and so on - according to [ISO week date definition](https://en.wikipedia.org/wiki/ISO_week_date)). I'd like to share with you what I've learnt from this task.

**TL;DR** - Use `ExtractIsoYear` function instead of `ExtractYear` when defining a year according to ISO-8601.

## Data model
Let's introduce a simple model that I'll using throughout this post - `Donation`.

{{< highlight python >}}
from django.db import models


class Donation(models.Model):
    amount = models.DecimalField(decimal_places=2, max_digits=10)
    created_at = models.DateTimeField()

    def __str__(self):
        return f"{self.created_at.isoformat()} {self.amount}$"
{{< /highlight >}}

Let's say that I'd like to perform some kind of aggregation per year of the week - for example I'm trying to answer a question like `I'd like to know how many donations were made in each week of the year`.


`created_at` field defines creation timestamp for a donation and I'll use it for further calculations.

Creating some exemplary objects:
{{< highlight python >}}
from datetime import datetime
from decimal import Decimal

import pytz

from someapp.models import Donation


Donation.objects.create(
    amount=Decimal("5.00"),
    created_at=datetime(2021, 1, 1, 10, 00, 00, tzinfo=pytz.UTC),
)
Donation.objects.create(
    amount=Decimal("2.00"),
    created_at=datetime(2021, 1, 2, 10, 00, 00, tzinfo=pytz.UTC),
)
Donation.objects.create(
    amount=Decimal("3.25"),
    created_at=datetime(2021, 1, 3, 10, 00, 00, tzinfo=pytz.UTC),
)
Donation.objects.create(
    amount=Decimal("14.00"),
    created_at=datetime(2021, 1, 4, 10, 00, 00, tzinfo=pytz.UTC),
)
Donation.objects.create(
    amount=Decimal("14.00"),
    created_at=datetime(2021, 1, 5, 10, 00, 00, tzinfo=pytz.UTC),
)
{{< / highlight >}}

## Extracting week & year from timestamp

To do the aggregation, I'll need to derive week & year from the `created_at` field. Quick peek at `django.db.models.functions` module and `ExtractWeek` & `ExtractYear` functions look like a good candidates for a job. Annotation would look like this:

{{< highlight python >}}
from django.db.models.functions import ExtractWeek, ExtractYear


Donation.objects.annotate(
    year=ExtractYear("created_at"), week=ExtractWeek("created_at"),
)
{{< / highlight >}}

But when you take a deeper look at aggregations, you'll notice something's wrong:
{{< highlight python >}}
Donation.objects.annotate(
    year=ExtractYear("created_at"), week=ExtractWeek("created_at"),
).values("year", "week").distinct()

# above expression returns
<QuerySet [{'year': 2021, 'week': 1}, {'year': 2021, 'week': 53}]>
{{< / highlight >}}

Year 2021 & week 53 seems like a future considering that max datetime of donation is January 5th, 2021. Why is that happening?

## ExtractIsoYear 

Turns out that, according to ISO-8601 definition, dates `2021-01-01`, `2021-01-02`, `2021-01-03` belong to the last week of **2020** (week number 53). `ExtractWeek` utilize the same ISO definition, while `ExtractYear` returns exact year from given timestamp.

To fix this mismatch, we'll need to use `ExtractIsoYear` function from `django.db.models.functions` module:

{{< highlight python >}}
from django.db.models.functions import ExtractWeek, ExtractIsoYear

Donation.objects.annotate(
    year=ExtractIsoYear("created_at"), week=ExtractWeek("created_at"),
).values("year", "week").distinct()

# result
<QuerySet [{'year': 2020, 'week': 53}, {'year': 2021, 'week': 1}]>
{{< / highlight >}}

Week 53 of 2020 and week 1 of 2021 is a correct result.

Hope this short post will help you to avoid some bugs in the future,<br>

Best Regards,

Kuba

