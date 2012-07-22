logistic-regression-sgd-mapreduce
=================================

# Overview
Python scripts for building binary classifiers using logistic regression with stochastic gradient descent, packaged for use with map-reduce platforms supporting Hadoop streaming.

# ￼Algorithm
* Distributed regularized binary logistic regression with stochastic gradient descent [[1]], [[2]]
	* Competitive with best extant large-scale supervised learning algorithms
	* Results provide direct estimation of probability of class membership 
* Implementation supports large-scale learning using Hadoop streaming


    $ cd logistic-regression-sgd-mapreduce

# Data formats

## Instances
    <instance> ::= { "class": <class>, "features": { <feature>: <value>, … , <feature>: <value> } }
    <class>    ::= 0 | 1
    <feature>  ::= a JSON string
    <value>    ::= a JSON float in the interval [0.0, 1.0]   

## Models
    <model> ::= { <feature>: <weight>, … , <feature>: <weight> }
    <weight> ::= a JSON float in the interval [0.0, 1.0]

## Confusion matrices
    <confusion_matrix> ::= { "TP": <count>, "FP": <count>, "FN": <count>, "TN": <count> }
    <count>            ::= a JSON int in the interval [0, inf)
    
# Usage

## From a UNIX shell
For small data sets, the scripts can be run from the command line.

### Convert data in SVM<sup><i>Light</i></sup> format into instances
Convert a file with data in SVM<sup><i>Light</i></sup> [[8]] format into a file containing instances. Awk, sort and cut are used here to randomly shuffle the data set, which is required to correctly train the model.

    $  cat train.data.svmlight | ./parse_svmlight_examples.py | awk 'BEGIN{srand();} {printf "%06d %s\n", rand()*1000000, $0;}' | sort -n | cut -c8- > train.data
    $  cat test.data.svmlight | ./parse_svmlight_examples.py | awk 'BEGIN{srand();} {printf "%06d %s\n", rand()*1000000, $0;}' | sort -n | cut -c8- > test.data

### Train a model
Generate a model from training data.

    $ cat train.data | ./train_map.py | sort | ./train_reduce.py > /path/to/your/model
    
Hyperparameters can be optionally set as environment variables:

    $ export MU=0.002   # the regularization parameter
    $ export ETA=0.5    # the learning rate
    $ export N=100000   # the number of instances in the training set

### Test a model
Generate a confusion matrix based on running a model against test data. The location of the model is passed as an environment variable that is a valid file URL.

    $ export MODEL=file:///path/to/your/model
    $ cat test.data | ./test_map.py | sort | ./test_reduce.py > confusion-matrix
    
### Predict
Generate a tab-separated file containing an instance preceded by a margin-based certainty based on the estimated probability of the instance, sorted in increasing order of certainty.

    $ export MODEL=file:///path/to/your/model
    $ cat test.data | ./predict_map.py | sort | ./predict_reduce.py > predictions
    
## Using Elastic MapReduce
For large-scale data sets, the scripts can be run using Hadoop streaming in Elastic MapReduce. First, upload the Python scripts and data files to a bucket in S3.

### Train a model
    $ ./elastic-mapreduce --create --stream \
		--input s3n://path/to/your/bucket/train.data \
		--mapper s3n://path/to/your/bucket/train_map.py \
		--reducer s3n://path/to/your/bucket/train_reduce.py \
		--output s3n://path/to/your/bucket/model

### Test a model
    $ ./elastic-mapreduce --create --stream \
		--input s3n://path/to/your/bucket/train.data \
		--mapper s3n://path/to/your/bucket/train_map.py \
		--reducer s3n://path/to/your/bucket/train_reduce.py \
		--output s3n://path/to/your/bucket/confusion-matrix
		--cmdenv MODEL=https://s3.amazonaws.com/path/to/your/bucket/model/part-00000
		
### Predict
    $ ./elastic-mapreduce --create --stream \
		--input s3n://path/to/your/bucket/train.data \
		--mapper s3n://path/to/your/bucket/train_map.py \
		--reducer s3n://path/to/your/bucket/train_reduce.py \
		--output s3n://path/to/your/bucket/predictions
		--cmdenv MODEL=https://s3.amazonaws.com/path/to/your/bucket/model/part-00000
    
# References

[[1]] Cohen, W. Stochastic Gradient Descent. Downloaded from http://www.cs.cmu.edu/~wcohen/10-605/notes/sgd-notes.pdf. (2012).



[1]: http://www.cs.cmu.edu/~wcohen/10-605/notes/sgd-notes.pdf "Cohen, W. Stochastic Gradient Descent. Downloaded from http://www.cs.cmu.edu/~wcohen/10-605/notes/sgd-notes.pdf. (2012)."
[8]: http://svmlight.joachims.org/ "Joachims, T. SVM<sup><i>Light</i></sup> Support Vector Machine. Downloaded from http://svmlight.joachims.org/. (2008)."