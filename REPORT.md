# Hands-on L11: Report

**Name:** Pablo Garces
**Student ID:** 801377963
**Email:** <pgarces@charlotte.edu>

---

## Seed and commands

Seed used (your student ID): `801377963`

The commands you ran, in order. If you deviated from the steps in the README, say where and
why.

```bash
docker compose up -d

docker cp main.py spark-master:/opt/spark/work-dir/
docker exec -it spark-master /opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  /opt/spark/work-dir/main.py task1 /opt/spark/work-dir/shared 801377963
docker exec -it spark-master /opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  /opt/spark/work-dir/main.py task2 /opt/spark/work-dir/shared 801377963

# Both failed because numpy is missing in the image (see Problems), so I installed it.
docker exec -u root spark-master pip install numpy
docker exec -u root spark-worker-1 pip install numpy
docker exec -u root spark-worker-2 pip install numpy

docker exec -it spark-master /opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  /opt/spark/work-dir/main.py task1 /opt/spark/work-dir/shared 801377963
docker exec -it spark-master /opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  /opt/spark/work-dir/main.py task2 /opt/spark/work-dir/shared 801377963

find shared-folder/models/full -maxdepth 2

docker compose down
```

---

## Results

### Task 1: baseline and full pipeline

Paste the two lines printed by `evaluate()` (also in `shared-folder/output/task1/metrics`),
the five predictions shown and the "saved to" line.

```
seed 801377963: 5753 training rows, 1393 test rows
baseline                         RMSE   186.2   R2  0.234
+--------+---------------+-----+------------------+
|bedrooms|      room_type|price|        prediction|
+--------+---------------+-----+------------------+
|     1.0|   Private room|200.0|114.40775109993456|
|     3.0|Entire home/apt|250.0| 382.1333213547082|
|     1.0|   Private room|115.0|29.770435470179564|
|     2.0|Entire home/apt|220.0| 271.6921298685943|
|     2.0|Entire home/apt|200.0| 295.3756186097994|
+--------+---------------+-----+------------------+
only showing top 5 rows
full                             RMSE   174.0   R2  0.331
saved to /opt/spark/work-dir/shared/models/full
```

How many training and test rows did your seed give? The slides show 5,780 and 1,366 with
seed 42; why is yours different?

My seed gave 5,753 training rows and 1,393 test rows. It is different because `randomSplit`
uses the seed to randomly decide where each row goes, so a different seed puts different rows
in each set.

### Task 2: scoring with the saved model

Paste the stage list, the RMSE of the loaded model and the five predictions.

```
stages: ['StringIndexerModel', 'OneHotEncoderModel', 'VectorAssembler', 'LinearRegressionModel']
RMSE of the loaded model: 174.0
+----------------------+---------------+--------+-----+----------+
|neighbourhood_cleansed|      room_type|bedrooms|price|prediction|
+----------------------+---------------+--------+-----+----------+
|       Diamond Heights|   Private room|     1.0|200.0|     114.4|
|               Bayview|Entire home/apt|     3.0|250.0|     382.1|
|               Bayview|   Private room|     1.0|115.0|      29.8|
|        Bernal Heights|Entire home/apt|     2.0|220.0|     271.7|
|        Bernal Heights|Entire home/apt|     2.0|200.0|     295.4|
+----------------------+---------------+--------+-----+----------+
only showing top 5 rows
1391 predictions for 1393 test rows
```

---

## Questions

**1. Baseline versus full.** With seed 42 the lecture gets 221.6 / 0.152 for the baseline
and 220.6 / 0.160 for the full pipeline. What did you get, and how much did 97 more
features buy? The price goes from 10 to 10,000 dollars a night (the `summary()` in the
lecture): how does that affect an RMSE measured in dollars, and why do the numbers change
from seed to seed? Name one change to the pipeline you would try first (slides "Running
Example" with the model table, and "Tuning").

I got an RMSE of 186.2 and an R2 of 0.234 for the baseline, and 174.0 and 0.331 for the full
pipeline. So the 97 extra features lowered the error by about 12 dollars and raised the R2 by
almost 0.1, which is more than in the lecture, where it only improved by 1 dollar.

Since the price goes from 10 to 10,000 dollars, a few very expensive listings have a big effect
on the RMSE. RMSE squares every error before averaging, so one listing where the model misses
by thousands of dollars counts a lot more than many listings where it misses by a few dollars.
This is why the numbers change from seed to seed: the seed decides which listings end up in the
test set, so the RMSE depends on how many of those expensive listings land there.

I would try predicting the log of the price instead of the price, and then converting the
prediction back. In this way, the expensive listings are not going to weigh as much as they do now.

**2. What is in `shared-folder/models/full`?** List its folders (`find shared-folder/models/full
-maxdepth 2`) and say what each holds. Why do we save the PipelineModel rather than the
LinearRegressionModel alone? Which stages of the saved model are fitted models and which
are plain transformers?

```text
$ find shared-folder/models/full -maxdepth 2
shared-folder/models/full
shared-folder/models/full/stages
shared-folder/models/full/stages/0_StringIndexer_ef8d7f641d9f
shared-folder/models/full/stages/1_OneHotEncoder_e03aa7a460d7
shared-folder/models/full/stages/2_VectorAssembler_58c32ea62297
shared-folder/models/full/stages/3_LinearRegression_c419f80a4033
shared-folder/models/full/metadata
shared-folder/models/full/metadata/._SUCCESS.crc
shared-folder/models/full/metadata/part-00000-4d7d05f0-c0fc-496e-9c4f-a6525823ad86-c000.txt
shared-folder/models/full/metadata/.part-00000-4d7d05f0-c0fc-496e-9c4f-a6525823ad86-c000.txt.crc
shared-folder/models/full/metadata/_SUCCESS
```

We save the whole PipelineModel because the LinearRegressionModel only knows how to read
a `features` vector. To score new listings, the text columns have to be turned into numbers
and the vector has to be built exactly the same way as in training, with the same index for
every category. If we only saved the regression, we would have to rebuild those steps again.

The fitted models are the `StringIndexer`, which learned which number goes with each category,
the `OneHotEncoder`, which learned how many categories each column has, and the
`LinearRegression`, which learned the coefficients and the intercept. The `VectorAssembler` is
a plain transformer because it only puts columns together and learns nothing, which is why its
folder has no `data`.

**3. Only transformers at scoring time.** Task 2 ran no `fit()`. Compare the number of
predictions written with the number of test rows: if they differ, where did the rows go,
and which parameter of which stage decides it? What would you have to change in task 2 to
score a stream of listings instead of a Parquet file (slide "Scoring a Stream")?

Task 2 wrote 1,391 predictions for 1,393 test rows, so 2 rows are missing. They were dropped by
the `StringIndexer`, because of `handleInvalid="skip"`. This happened because those 2 listings
have a category that never appeared in the training set, so the indexer has no number for it
and skips the row instead of failing.

To score a stream, I would read the listings with `spark.readStream` instead of
`spark.read.parquet`, giving it the schema of the test set because a stream can't infer it. The
`model.transform` part stays the same, because the PipelineModel only has transformers. At the
end, instead of `write.csv`, I would use `writeStream`, which is the one made for streams.

---

## Problems and fixes

Anything that went wrong and what resolved it. Paste the actual error message. If nothing
went wrong, say so.

My first runs of task 1 and task 2 failed right at the start, so I used AI to help me figure
out why. It ended up being because the `apache/spark:4.2.0` image does not come with numpy,
and `pyspark.ml` needs it:

```text
  File "/opt/spark/work-dir/main.py", line 25, in <module>
    from pyspark.ml import Pipeline, PipelineModel
ModuleNotFoundError: No module named 'numpy'
```

I installed it in the master and in both workers with
`docker exec -u root <container> pip install numpy`, and after that both tasks ran correctly.
