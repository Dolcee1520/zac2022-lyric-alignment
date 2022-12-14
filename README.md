# Zalo AI Challenge 2022 - Lyric Alignment

This is a solution for track Lyric Alignment - From team VTS-HTML.

## Description

Details are at the link of the [ZaloAI - Lyric Alignment](https://challenge.zalo.ai/portal/lyric-alignment)

## Solutions 

### Overview

Our method is based on paper [Improving Lyrics Alignment through Joint Pitch Detection](https://arxiv.org/pdf/2202.01646.pdf) by Jiawen Huang, Emmanouil Benetos, Sebastian Ewer, (ICASSP). 2022.

The overview will be discussed below this figure

![](./figure/overview.png)

### Vocal Extraction

Realizing that in the ZaloAI dataset there are many songs with fast tempo and difficult to hear the vocals, so we decided to preprocess the audio with a **small module** [Vocal remover](https://github.com/tsurumeso/vocal-remover) by Tsurumeso. From now on, we have an audio file with the sound containing only clear vocals.

### Acoustic Model

The acoustic model takes the Mel-spectrogram of the separated vocals as input, and produces the phoneme posteriorgram.

It consists of a convolutional layer, a residual convolutional block, a fully-connected layer, 3 bidirectional LSTM (Long Short-Term Memory) layers, a final fullyconnected layer, and non-linearities in between. Model trained using CTC Loss.

### Alignment Process

The alignment process use Viterbi forced alignment. This can be computed efficiently via dynamic programming.

### Problem with datasets

First, we tried to fine-tuning this method on ZaloAI 2022 datasets, the results in the public leaderboard seem not good $\\approx 0.51$. After EDA data of training set, we realized that Ground-truth has been misalignment quite a bit (easily check this by [audacity](https://www.audacityteam.org/download/)).

So we used the [pretrained baseline](https://github.com/jhuang448/LyricsAlignment-MTL/blob/main/checkpoints/checkpoint_Baseline) to create **pseudo labels**. 

However, instead of using 100% pseudo-labels. We calculate IoU between pseudo-labels and GT from ZaloAI Datasets. If IoU is less than 0.7, we will use pseudo-label, otherwise we will use GT from Zalo. (approximate 1/3 total samples)

After training with this dataset, the results improved significantly $\\approx 0.57$. 

### Post-processing

The predicted results of our model are quite close to reality. However, if you pay close attention, GT has a characteristic that **the end time** of the previous word and **the start time** of the following word is the same. So we extend the gaps between the start and end times of two adjacent words into the previous word. This improves our results quite a bit $\\approx 0.60$.

```
pip install -r requirements.txt
```

Besides, you might want to install some source-separation tool (e.g. [Spleeter](https://github.com/deezer/Spleeter), [Open-Unmix](https://github.com/sigsep/open-unmix-pytorch))
or use your own system to prepare source-separated vocals.

## Usage

Check the [notebook](https://github.com/jhuang448/LyricsAlignment-MTL/blob/main/example.ipynb) for a quick example.

## Data

The **DALI v2.0** is required for training. See instructions on how to get the dataset: [https://github.com/gabolsgabs/DALI](https://github.com/gabolsgabs/DALI). 

To use the DALI data loader, it is recommended to pull the repo and link to the root of this repo by running:

```
ln -s path/to/dali_wrapper/ DALI
```

The annotated **Jamendo** is used for evaluation: [https://github.com/f90/jamendolyrics](https://github.com/f90/jamendolyrics) 

All the songs in both datasets need to be separated and saved in advance. 

When you run the training/testing scripts for the first time, hdf5 files will be generated.

## Training

### The baseline acoustic model (**Baseline**)

```
python train.py --dataset_dir=/path/to/DALI_v2.0/annotation/ --sepa_dir=/path/to/separated/DALI/vocals/ 
                --hdf_dir=/where/to/save/hdf5/files/
                --checkpoint_dir=/where/to/save/checkpoints/ --log_dir=/where/to/save/tensorboard/logs/ 
                --model=baseline --cuda
```

### The proposed acoustic model (**MTL**)

```
python train.py --dataset_dir=/path/to/DALI_v2.0/annotation/ --sepa_dir=/path/to/separated/DALI/mp3s/ 
                --hdf_dir=/where/to/save/hdf5/files/ --loss_w=0.5
                --checkpoint_dir=/where/to/save/checkpoints/ --log_dir=/where/to/save/tensorboard/logs/ 
                --model=MTL --cuda
```

Run `python train.py -h` for more options.

## Inference

The following script runs alignment using a pretrained baseline model without boundary information (**Baseline**) on Jamendo:

```
python eval.py --jamendo_dir=/path/to/jamendolyrics/ --sepa_dir=/path/to/separated/jamendo/mp3s/
               --load_model=./checkpoints/checkpoint_Baseline --pred_dir=/where/to/save/predictions/
               --model=baseline
```

The following script runs alignment using the pretrained MTL model with boundary information (**MTL+BDR**) on Jamendo:

```
python eval_bdr.py --jamendo_dir=/path/to/jamendolyrics/ --sepa_dir=/path/to/separated/jamendo/mp3s/
                   --load_model=./checkpoints/checkpoint_MTL --pred_dir=/where/to/save/predictions/
                   --bdr_model=./checkpoints/checkpoint_BDR --model=MTL
```

The generated csv files under `pred_dir` can be easily evaluated using the evaluation script in [jamendolyrics](https://github.com/f90/jamendolyrics).

## References

[1] Yun-Ning Hung, Yi-An Chen, and Yi-Hsuan Yang, “Multi-task learning for frame-level instrument recognition,” in Proc. ICASSP. 2019, pp. 381–385, IEEE.

[2] Sebastian Ewert, Meinard Müller, and Peter Grosche, “High resolution audio synchronization using chroma onset features,” in Proc. ICASSP. 2009, pp. 1869–1872, IEEE.

[3] Daniel Stoller, Simon Durand, and Sebastian Ewert, “End-to-end lyrics alignment for polyphonic music using an audio-to-character recognition model,” in Proc. ICASSP. 2019, pp. 181–185, IEEE.

[4] Gabriel Meseguer-Brocal, Alice Cohen-Hadria, and Geoffroy Peeters, “Creating DALI, a large dataset of synchronized audio, lyrics, and notes,” Transactions of the International Society for Music Information Retrieval, vol. 3, no. 1, pp. 55–67, 2020.

[5] Chitralekha Gupta, Emre Yılmaz, and Haizhou Li, “Automatic lyrics alignment and transcription in polyphonic music: Does background music help?,” in Proc. ICASSP. 2020, pp. 496–500, IEEE.


## Contact

Jiawen Huang

jiawen.huang@qmul.ac.uk
