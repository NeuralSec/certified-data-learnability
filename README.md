## The code of our paper "Provably Unlearnable Data Examples" (NDSS'25 Distinguished Paper Award)

<p align="center">
  <a href="https://neuralsec.github.io/projects/provably-unlearnable-data-examples/">
    <img src="https://img.shields.io/badge/Project-Page-blue?style=for-the-badge" alt="Project Page">
  </a>
  <a href="https://www.ndss-symposium.org/wp-content/uploads/2025-886-paper.pdf">
    <img src="https://img.shields.io/badge/Paper-PDF-red?style=for-the-badge" alt="Paper">
  </a>
</p>

In this paper, we introduce the concept of certified learnability.
Certified $(q,\eta)$-Learnability measures how learnable a dataset is by computing a probabilistic upper bound on the test performance of classifiers trained on this dataset, as long as those classifiers fall within a certified parameter set.
We use certified $(q,\eta)$-Learnability as a measurement of the effectiveness and robustness of unlearnable examples, and propose Provably Unlearnable Examples (PUEs) which can lead to reduced $(q,\eta)$-Learnability when training classifiers on them.

---
## **1. Environment setup**

**Setup virtual environment and install necessary packages:**

Required Python version:
<div align="left">
  
  <a>![Python 3.11.4](https://img.shields.io/badge/Python-3.11.4-brightgreen.svg)</a>
  
</div>

```
python -m venv pue-env
source pue-env/bin/activate
cd $repository_name
pip install --upgrade pip
pip install -r requirements.txt --no-cache-dir
```

**Download datasets:**\
CIFAR-10 and CIFAR-100 datasets will be automatically downloaded.\
Download the ImageNet-100 dataset from [Hugging Face](https://huggingface.co/datasets/jokerak/imagenet100/resolve/main/imagenet-100.zip) using the following script.
```
bash download_imagenet100.sh
```

---

## **2. ```certify``` the learnability via a surrogate trained on unlearnable examples**
A script ```certify.sh``` is provided for $(q,\eta)$-Learnability certification.
This script lets you certify the learnability of a dataset based on a surrogate model trained on this dataset.

The path to the surrogate weights trained on the unlearnable examples you want to certify should be provided to the ```surrogate_path``` variable.
The example shell commands in the script are:
```
#!/bin/bash -l

# Activate virtual environment
source ../pue-env/bin/activate

# Define variables
export perturb_type=classwise
export noise_type=PUE
export base_version=resnet18
export config_path=configs/cifar10
export data_type=CIFAR10
export attack_type=min-min
export N=1000
export u_p=1
export sigma=0.25
export q=0.9
export train_noise=0.25

# Path the experiment name and the path to the PUE surrogate weights
export exp_name=./cert_exp/${perturb_type}/${data_type}/${noise_type}/train_noise=${train_noise}/u_p=${u_p}/q=${q}-sigma=${sigma}
export surrogate_path=./checkpoints/online_surrogate/${noise_type}/${perturb_type}/${data_type}/noise=${train_noise}-u_p=${u_p}

# Provide the experiment name and the path to the baseline surrogate weights
#export exp_name=./cert_exp/${perturb_type}/${data_type}/${noise_type}/q=${q}-sigma=${sigma}
#export surrogate_path=./checkpoints/online_surrogate/${noise_type}/${perturb_type}/${data_type}

# Certify
python certify.py --exp_name ${exp_name} \
                  --config_path ${config_path} \
                  --surrogate_path ${surrogate_path} \
                  --train_data_type ${data_type} \
                  --test_data_type ${data_type} \
                  --version ${base_version} \
                  --perturb_type ${perturb_type} \
                  --sigma ${sigma} \
                  --q ${q} \
                  --N ${N} \
                  --data_parallel
```

**Adjustable key parameters:**\
```q``` -- The value of $q$ in $(q,\eta)$-Learnability. \
```sigma``` -- The standard deviation of Gaussian parametric noise in the certification.\
```N``` -- The number of Gaussian parametric noise draws in the certification algorithm. ```N=1000``` by default.

---

## **3. Generate PUEs**
A script ```make_pue.sh``` is provided for generating PUEs and an online certification surrogate.
The example shell commands are:

```
#!/bin/bash -l

# Activate virtual environment
source ../pue-env/bin/activate

# Define variables
export perturb_type=classwise
export base_version=resnet18
export config_path=configs/cifar100
export data_type=CIFAR100
export attack_type=min-min
export num_steps=1
export universal_stop_error=0.1
export epsilon=8
export step_size=0.8
export train_step=20
export robust_noise=0.25
export u_p=10
export universal_train_target='train_subset'
export exp_name=./pue_exp/${perturb_type}/${data_type}/noise=${robust_noise}-u_p=${u_p}

# Train an online surrogate and generate PUEs
python make_pue.py    --config_path          ${config_path} \
                      --train_data_type      ${data_type} \
                      --test_data_type       ${data_type} \
                      --version              ${base_version} \
                      --noise_shape          100 3 32 32 \
                      --epsilon              ${epsilon} \
                      --num_steps            ${num_steps} \
                      --step_size            ${step_size} \
                      --train_step           ${train_step}     \
                      --attack_type          ${attack_type} \
                      --perturb_type         ${perturb_type} \
                      --robust_noise         ${robust_noise}\
                      --avgtimes_perturb     ${u_p}  \
                      --exp_name             ${exp_name} \
                      --universal_stop_error ${universal_stop_error} \
                      --data_parallel \
                      --universal_train_target ${universal_train_target} \
                      --use_subset \
                      --pue
```

**Adjustable key parameters:**\
```base_version``` -- The classifier architecture to use (selecting from {```resnet18, resnet50, dense121```}).\
```data_type``` -- Specifying a training dataset.\
```--noise_shape``` -- Shape of the perturbation. It should match the shape of the dataset to be protected.\ 
```config_path``` -- Path to the corresponding config file.\
```num_steps``` -- The steps of perturbation optimization in each perturbation optimization iteration.\
```universal_stop_error``` -- Stop training when validation error is reduced below this value.\
```epsilon``` -- Perturbation magnitude (```epsilon/255```) in $\ell_{\infty}$ norm.\
```step_size``` -- Step size for optimizing the perturbation.\
```train_step``` -- Iteration number for optimizing the perturbation.\
```robust_noise``` -- Train-time noise scale.\
```u_p``` -- Number of random weight draws in perturbation optimization.

---

## **4. Launch a recovery attack against a classifier trained on PAP noise**

```
#!/bin/bash -l

# Activate virtual environment
source ../pue-env/bin/activate

export config_path=configs/cifar100
export data_type=CIFAR100  #{CIFAR10, CIFAR100}
export method=PUE
export train_mode=normally_trained #{online_surrogate, normally_trained}
export recovery_rate=0.2

# Path to trained classifiers 
export surrogate_path=./checkpoints/${train_mode}/${method}/classwise/${data_type}

# Path to PUE surrogate:
#export surrogate_path=./checkpoints/${train_mode}/${method}/classwise/${data_type}/noise=0.25-u_p=10

for eta_loop in 0.2 0.4 0.6 0.8 1.0 1.2 1.4 1.6 1.8 2.0
do
export eta=${eta_loop}
# Train the model
python recover.py  --version                 resnet18_recover \
                        --model_version           resnet18 \
                        --exp_name                recover_exp_best/${method}/${data_type}/recovery_rate=${recovery_rate}/eta=${eta} \
                        --config_path             ${config_path} \
                        --train_data_type         ${data_type} \
                        --test_data_type          ${data_type} \
                        --train_batch_size        128 \
                        --eta                     ${eta} \
                        --recover_rate            ${recovery_rate}\
                        --surrogate_path          ${surrogate_path} \
                        --load_model   \
                        --project      
                        #--use_train_subset   
done
```

**Adjustable key parameters:**\
```use_train_subset``` -- Use training subset in the recovery attack if ```--use_train_subset``` is turned on. Otherwise, use the test set for the attack.\
```recovery_rate``` -- The proportion of training data used in the recovery attack.


---

## Extra Results on the Validity of the Certification
We train 10 ResNet-18 models on CIFAR10 PUEs with random weight initialization, batch loading orders, and data augmentations, and record the weight distributions in ten layers drawn from each model. The results are demonstrated in the following figure. Each row contains the parameter distributions in the same layer from the 10 ResNet-18 models.

![Figure1](./resources/layer-conv1.weight-weights-distributions.png)
![Figure2](./resources/layer-layer1.0.conv1.weight-weights-distributions.png)
![Figure3](./resources/layer-layer1.1.conv1.weight-weights-distributions.png)
![Figure4](./resources/layer-layer2.0.conv1.weight-weights-distributions.png)
![Figure5](./resources/layer-layer2.1.conv1.weight-weights-distributions.png)
![Figure6](./resources/layer-layer3.0.conv1.weight-weights-distributions.png)
![Figure7](./resources/layer-layer3.1.conv1.weight-weights-distributions.png)
![Figure7](./resources/layer-layer4.0.conv1.weight-weights-distributions.png)
![Figure7](./resources/layer-layer4.1.conv1.weight-weights-distributions.png)
![Figure7](./resources/layer-linear.weight-weights-distributions.png)

The distribution of pairwise parameter difference across the ten ResNet-18 is as follows:

<img src="./resources/Param-diff-distribution.png" alt="drawing" width="500"/>

The converged weights from different stochastic training runs have a mean parameter difference of $-4.92\times 10^{-6}$ and an STD of $0.01$, 
Since the certified parameter set with $\sigma=0.25$ and $\eta=1.0$ has sufficiently large probability mass within $[\hat{\theta}-0.5, \hat{\theta}+0.5]$ ($\hat{\theta}$ is the set of parameters of the surrogate), the results suggest that the certified parameter set verified using one of these models as a surrogate can maintain its coverage over classifier weights from other training runs with stochasticity. 
The certified parameter set can effectively capture classifiers trained separately by adversaries following certain standard training procedures.

---

## Acknowledgments

The code used in this work is inherited from the following repository:

[https://github.com/HanxunH/Unlearnable-Examples](https://github.com/HanxunH/Unlearnable-Examples)

---

## Citation

Please cite the repository as follows if you find it useful.
```
@inproceedings{wang2025provably,
  title={Provably Unlearnable Data Examples},
  author={Wang, Derui and Xue, Minhui and Li, Bo and Camtepe, Seyit and Zhu, Liming},
  booktitle={The Network and Distributed System Security (NDSS) Symposium},
  year={2025}
}
```
