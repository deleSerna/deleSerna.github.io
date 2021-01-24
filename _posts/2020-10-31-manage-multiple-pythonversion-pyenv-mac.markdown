---
layout: post
title:  "How to manage multiple version of python in mac"
date:   2020-10-31 9:32:36 +0530
categories: Python pyenv mac homebrew
---
When it comes to programming language, Java is my goto langauge. But when I would like to try some sample realed to NLP or machine learning, I will compelled to use
python because most of the examples were in python. As I follow the examples, I will try to do pip install as said in the examples. When I started doing different
categories of examples (spacy, rasa, scikit etc..), I found that different libaries were depending on different version of libary and I messed up all my python installation with
different version of those libary. Then I realized that it's easier to use different python environment for different type of problems. Then I came to read a wonderful article about 'pyenv'
and started using that. As I used pyenev further, I realized that eventhough I screwup that particular version of python it does not screw up all the system
and I can live with other version peacefully. I could not find the original reference which I read that time hence here I can not point to that. Here I will be some other
refer some other source which I found just before installing pyenv in a new system [1]. All the following commands are for mac os. 

1. Install brew, if it's not already installed
```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
```
2. Install pyenv
``` brew install pyenv```
3. Install pyenv-virtualenv
``` brew install pyenv-virtualenv ```
4. Once the installation completed, please use following command to see the avaialable python versiona
``` pyenv install --list ```
5. Install some version which you would like to work on by specify that version
``` pyenv install 3.9.0```
6.  To create a virtualenv for the desire python version
``` pyenv virtualenv 3.9.0 your-virtual-env-name```
7. You can see the created your-virtual-env-name by 
``` ls ~/.pyenv/versions ```
8. Open ~/.bash_profile and paste following commands, otherwise when we try to activate the create virtual environment we will get the following issue 
" pyenv-virtualenv has not been loaded into your shell properly"
  ```eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"
```
9. Run bashrc by
``` source  ~/.bash_profile ```

10 Now activate your virtual env by
``` pyenv activate your-virtual-env-name ```

11 If you want to remove your virtual env the you can simply remove the folder in ~/.pyenv/versions/virtual-nev-name


**References**
1. [https://levelup.gitconnected.com/how-to-set-up-python-environment-on-your-mac-560ebf9324ed][ref-1]
