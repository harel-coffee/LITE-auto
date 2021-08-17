LITE is an auto-tuning system for various Spark applications on large-scale datasets. 

# Performance Overview
## Tuning performance on Sparkbench applications
We have conducted comparative experiments on 15 spark-bench applications. 

![*Figure 1 Percentage of execution time reduction by different tuning methods*](https://github.com/cheyennelin/LITE/blob/main/fig1.png)

Figure 1 presents comparative performance of LITE, against several baselines. 
(1) Default: using Spark's default configurations. 

(2) Manual: manually tuning the applications based on expertise and the online tuning guides for maximally 12 hours. 

(3) BO: trial-and-error Bayesian Optimization tuning, where Gaussian Process was the surrogate model and Expected Improvement was the acquisition function. We used 5 most similar instances in the training set to initialize Gaussian Process. Then, BO was trained for at least 2 hours. 

(4) DDPG: a reinforcement learning framework based on Deep Deterministic Policy Gradient, where the action space was the configurations, and the state was the inner status summary of Spark. DDPG was also trained for at least 2 hours.

(5) DDPG+Code: similar as QTune, an additional module is incorporated in DDPG, which predicts the change of outer metric based on pretrained code features extracted by LITE and the inner status.

(6) RFR: a Random Forest Regression model is trained to map the input datasize and the application to an appropriate knob value. Since the prediction of RFR is numerical, for discrete valued knobs, we round the prediction of RFR to the nearest integer. 

Figure 1 reports percentage of execution time reduction, which is defined as t/tmin, where t is the execution time produced by the method, and tmin is the smallest execution time by different methods on this application, according to Table 1. 

*Table 1 Actural execution time by different methods*
| Application | PCA  | CC     | DT   | KM    | LP   | LR   | Logit | PR   | PO   | SP   | SCC  | SVD++ | SVM    | TS   | TC   |
|-------------|------|--------|------|-------|------|------|-------|------|------|------|------|-------|--------|------|------|
| Default     | 3600 | 7200   | 5578 | 18756 | 7200 | 7141 | 2649  | 7200 | 7200 | 7200 | 7200 | 7200  | 7200   | 7200 | 7200 |
| Manual      | 408  | 304    | 498  | 2655  | 413  | 1283 | 324   | 4099 | 253  | 217  | 515  | 445   | 665    | 667  | 7200 |
| DDPG(2h)    | 1396 | 98     | 523  | 3288  | 349  | 3476 | 1126  | 2553 | 395  | 7200 | 1095 | 7200  | 3600   | 1131 | 114  |
| DDPG+Code   | 342  | 7200   | 547  | 3113  | 572  | 971  | 617   | 7200 | 7200 | 1276 | 443  | 423   | 1443   | 2030 | 83   |
| BO(2h)      | 339  | 84     | 737  | 2884  | 353  | 614  | 619   | 7200 | 249  | 168  | 586  | 423   | 675    | 1865 | 81   |
| RFR         | 498  | 720    | 1380 | 2884  | 7200 | 588  | 480   | 7200 | 7200 | 7200 | 720  | 1560  | 660    | 336  | 7200 |
| LITE        | 316  | 81.933 | 449  | 2881  | 348  | 448  | 345   | 2184 | 116  | 145  | 316  | 352   | 456.97 | 325  | 65   |
| tmin        | 316  | 81.933 | 449  | 2655  | 348  | 448  | 324   | 2184 | 116  | 145  | 316  | 352   | 456.97 | 325  | 65   |

In summary, the average actural execution time and average percentage of execution time reduction for different tuning methods are:

|                                                | Default     | Manual      | DDPG(2h)    | DDPG+Code   | BO(2h)      | RFR         | LITE     |
|------------------------------------------------|-------------|-------------|-------------|-------------|-------------|-------------|----------|
| Average Actural Execution Time                 | 7314.933333 | 1329.733333 | 2236.266667 | 2230.666667 | 1125.133333 | 3055.066667 | 588.594  |
| Average Percentage of Execution Time Reduction | 0.074       | 0.626       | 0.442       | 0.496       | 0.689       | 0.410       | 0.991    |


# LITE 
## Enviroment

The project is implemented using python 3.6 and tested in Linux environment. Our system environment and cuda version as follows:

```
ubuntu 18.04
Hadoop 2.7.7
spark 2.4.7
HiBench 7.0
JAVA 1.8
SCALA  2.12.10
Python 3.6
Maven 4.15
CUDA Version: 10.1
```

We get the training data by running the workload in SparkBench，which can be installed by referring to https://github.com/CODAIT/spark-bench.

## 1. Data Generate


You can generate log files using the following command:

```
python scripts/bo_sample.py <path_to_spark_bench_folders>
```

The log file will be saved on your spark history server, you can use the following command to download it to the local.

```
hdfs dfs -get <path_to_spark_history_server_folders_on_hdfs> <local_folders>
```

## 2. Process Original Log
**scripts/history_by_stage.py** is used to parse the original log file generated by spark-bench, which get the running time of each stage, the amount of data read and write, etc.

```
python scripts/history_by_stage.py <spark_bench_conf_path> <history_dir> <result_path>
```

Note that the log file does not contain the data volume features of the workload, we need to add the data volume features through the configuration file in <spark_bench_conf_path>, result is like:

```json
{
	"AppId": "application_1616731661908_1527",
	"AppName": "SVM Classifier Example",
	"Duration": 23656,
	"SparkParameters": ["spark.default.parallelism=6", "spark.driver.cores=6"......],
	"StageInfo": {
		"0": {
			"duration": 3234,
			"input": 134283264,
			"output": 0,
			"read": 0,
			"write": 0
		}......
	},
	"WorkloadConf": ["NUM_OF_EXAMPLES=1600000", "NUM_OF_FEATURES=100"]
}
```

After the above steps, the generated data is still based on workload, and you need to use **scripts/build_dataset.py** to divide each row into multiple stages. Then merge the data of all stages of all workloads into a complete data set and store it in csv format. The data set can be divided into training set and test set as needed.

```
python scripts/build_dataset.py <result_path> <dataset_path>
```


## 3.Data Preprocessing

Obtain the stage code characteristics: enter the instrumentation folder, maven it into a jar package which name is preMain-1.0.jar 
```
cd instrumentation 
mvn clean package
```
and add  the package to the spark-submit's shell file as follow:
```sh
spark-submit --class <workload_class> --master yarn --conf "spark.executor.cores=4"  --conf "spark.executor.memory=5g" --conf "spark.driver.extraJavaOptions=-javaagent:<path_to_your_instrumentation_jar>/preMain-1.0.jar" <path_to_spark_bench>/<workload>/target/spark-example-1.0- SNAPSHOT.jar
```

The stage code is in /inst_log, you can change it by yourself; The file you get by instrumentation should be parsed by prediction_ml/spark_tuning/by_stage/instrumentation/all_code_by_stage/[get_all_code.py](https://github.com/cheyennelin/LITE/blob/main/prediction_ml/spark_tuning/by_stage/instrumentation/all_code_by_stage/get_all_code.py)

```
python get_all_code.py <folder_of_instrumentation> <code_reuslt_folder>
```

Our model is saved in [prediction_nn](https://github.com/cheyennelin/LITE/tree/main/prediction_nn),

You should use **data_process_text.py、dag2data.py、dataset_process.py** to get dictionary information、Process the edge and node information of the graph, and Integrate all features.

```
python data_process_text.py <code_reuslt_folder>
python dag2data.py <log_folder>
python dataset_process.py
```

Dictionary information and graph information  are saved in [dag_data](https://github.com/cheyennelin/LITE/tree/main/prediction_nn/dag_data), all these information and the dataset in section 2 will be Integrate by [dataset_process.py](https://github.com/cheyennelin/LITE/blob/main/prediction_nn/dataset_process.py), the final dataset will in the folder dataset.

## 4.Model Training

Then use **fast_train.py** to train model.

```
python fast_train.py
```

You can change the config of model through [config.py](https://github.com/cheyennelin/LITE/blob/main/prediction_nn/config.py), and the model will be saved in the folder model_save.

You can also use **trans_learn.py** finetune the model.

```
python trans_learn.py
```

## 5.Model Testing

[nn_pred_1.py](https://github.com/cheyennelin/LITE/blob/main/prediction_nn/nn_pred_1.py) can test the model，we use [predict_first_cold.py](https://github.com/cheyennelin/LITE/blob/main/prediction_nn/predict_first_cold.py) to predict the best combination of parameters and check its actual operation.

```
python nn_pred_8.py
python predict_first_cold.py
```
