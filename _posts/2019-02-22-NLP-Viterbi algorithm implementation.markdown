---
layout: post
title:  "NLP Viterbi Algorithm Implementation"
date:   2019-02-22 17:45:36 -0400
thumbnail: "/image/viterbi/6_w_rare_group.PNG"
categories: projects
---

```python
#!/usr/bin/python3

__author__="Sin-Yi Huang"
__date__ ="$Feb 20, 2019"

import io
import numpy as np
from math import log
import time

MIN = float(-99999)


def load_word_tag_count():
    word_tag = {}  # {'word': {'tag1': count1, 'tag2': count2, 'tag3': count3]}
    tag_count = {}  # {'tag': count} for tags in training data
    gram_count = {}  # {'w,u,v': count}

    with open("../data/ner_rare_group.counts", 'r') as f:
        lines = f.readlines()
        for line in lines:
            line = line.strip()  # remove '\n'
            line = line.split(" ")
            if line[1] == "WORDTAG":
                count = float(line[0])
                tag = line[2]
                word = line[3]

                if word in word_tag:  # if this word is in the training data
                    if tag not in word_tag[word]:
                        word_tag[word][tag] = count  # add new tag to this word
                else:  # if this word is not in the dict
                    word_tag[word] = {tag: count}  # add it to the dictionary

                if tag in tag_count:  # if the tag is in the dict
                    tag_count[tag] += count  # update tag's count
                else:  # if the tag is not in the tags dictionary
                    tag_count[tag] = count  # add it to the dictionary

            else:
                if line[1] == "1-GRAM":  # 10001 1-GRAM I-ORG
                    gram_count[line[2]] = line[0]
                elif line[1] == "2-GRAM":
                    gram_count[",".join(line[-2:])] = line[0]  # 1349 2-GRAM * I-PER
                else:  # trigram
                    gram_count[",".join(line[-3:])] = line[0]  # 883 3-GRAM * I-ORG I-ORG
    return word_tag, tag_count, gram_count


def get_trigram_log_prob(trigram, bigram, gram_count):
    if trigram in gram_count:
        numerator = float(gram_count[trigram])
    else:
        return MIN

    if bigram in gram_count:
        denominator = float(gram_count[bigram])
    else:
        return MIN

    return log(numerator / denominator)


def viterbi_w_bp(snt, word_tag, tag_count, gram_count):
    n = len(snt)
    star = ['*']
    tags = list(tag_count)  # ['I-LOC', 'B-ORG', 'I-PER', 'O', 'I-MISC', 'B-MISC', 'I-ORG', 'B-LOC']
    pi = {}
    pi[(0, '*', '*')] = 0
    bp = {}

    # generate pi, bp
    for k in range(1, n+1):  # k = 1~n ; if n=1, k = 1
        word = snt[k-1]

        if (k-2) < 1:
            ws = star
        else:
            ws = tags
        if (k-1) < 1:
            us = star
        else:
            us = tags
        vs = tags

        for u in us:
            for v in vs:
                #  calculate emission log probability
                if word in word_tag:  # key = word
                    if v in word_tag[word]:  # key = tag
                        emission_log_prob = log(word_tag[word][v] / tag_count[v])

                    else:  # this word doesn't has this tag
                        emission_log_prob = MIN
                else:  # unseen word
                    # _rare_
                    if word.isupper():
                        new_word = "_UPPER_"
                    elif word.isdigit():
                        new_word = "_DIGIT_"
                    elif word.istitle():
                        new_word = "_TITLE_"
                    elif word.islower():
                        new_word = "_LOWER_"
                    else:
                        new_word = "_RARE_"
                    if v in word_tag[new_word]:
                        emission_log_prob = log(word_tag[new_word][v] / tag_count[v])
                    else:
                        emission_log_prob = MIN

                pi_list = []

                for w in ws:
                    trigram = w +','+ u +','+v
                    bigram = w +','+ u

                    trigram_log_prob = get_trigram_log_prob(trigram, bigram, gram_count)

                    tmp_pi = pi[(k-1, w, u)] + trigram_log_prob + emission_log_prob

                    pi_list.append(tmp_pi)

                max_idx = np.argmax(np.asarray(pi_list))
                pi[(k, u, v)] = pi_list[max_idx]
                bp[(k, u, v)] = ws[max_idx]

    # generate y[n-1], y[n]
    pi_list = []
    uv_list = []
    y = ["*"] * (n + 1)
    for u in tags:
        for v in tags:
            trigram = u + ',' + v + ',STOP'
            bigram = u + ',' + v

            trigram_log_prob = get_trigram_log_prob(trigram, bigram, gram_count)

            if (n, u, v) in pi: # if pi[(n, u, v)] exists
                tmp_pi = pi[(n, u, v)] + trigram_log_prob
            else:
                tmp_pi = MIN + trigram_log_prob

            pi_list.append(tmp_pi)
            uv_list.append({'u': u, 'v': v})

    max_idx = np.argmax(np.asarray(pi_list))

    y[n-1] = uv_list[max_idx]['u']
    y[n]   = uv_list[max_idx]['v']

    # special case: if sentence length is one
    return_pi = []
    if n <= 1:
        return_pi = [pi_list[max_idx]]
        return y[1:], return_pi

    else:
        for i in range(2, n): # i = 2 ~ n-1
            k = n-i  # k = n-2 ~ 1
            y[k] = bp[(k+2, y[k+1], y[k+2])]

        for i in range(1, n+1):
            try:
                return_pi.append(pi[(i, y[i-1], y[i])])  # prob for word at k position = pi[(k, y[k-1], y[k])]
            except:
                return_pi.append(0)
    return y[1:], return_pi


def run():
    #  read log probability of trigram in training data
    word_tag, tag_count, gram_count = load_word_tag_count()

    with open("../data/ner_dev.dat", 'r') as r, open("../output/6.txt", 'w', io.DEFAULT_BUFFER_SIZE) as w:
        snt = []
        lines = r.readlines()
        for line in lines:
            word = line.strip()  # remove '\n'
            if word is "":  # sentence end
                tags, max_probs = viterbi_w_bp(snt, word_tag, tag_count, gram_count)
                for i in range(len(snt)):
                    w.write(snt[i] + " " + tags[i] + " " + str(max_probs[i]) + "\n")
                w.write("\n")
                snt = []
                continue
            snt.append(word)
        w.flush()

start = time.time()
run()
end = time.time()

#print("Execution time:", end-start)
```
#### About this code:
This code using not only "_RARE_" to replace words that appear < 5 times. It also enhenced by using other specific tag for words with special characteristic like: "_UPPER_", "_DIGIT_", "_TITLE_", "_LOWER_". It shows improvement in F1 score.

#### F1 score without any enhancement:
![then](/image/viterbi/5_2_wo_rare.PNG)

#### F1 score with only "_RARE_" tag:
![then](/image/viterbi/5_2_w_rare.PNG)

#### F1 score with "_RARE_", "_UPPER_", "_DIGIT_", "_TITLE_", "_LOWER_" tags:
![then](/image/viterbi/6_w_rare_group.PNG)

#### Viterbi:
- Both 5_2.py, 6.py are viterbi taggers, the difference is 5_2.py only used one symbol to represent unseen words but 6.py used 4 more kinds of symbol to represent.
- Pseudo code of viterbi(p.4) [http://www.cs.columbia.edu/~mcollins/cs4705-spring2019/hw1/h1_spring2019.pdf](http://www.cs.columbia.edu/~mcollins/cs4705-spring2019/hw1/h1_spring2019.pdf)
- More informaion: [http://www.cs.columbia.edu/~mcollins/cs4705-spring2019/hw1/h1_spring2019.pdf](http://www.cs.columbia.edu/~mcollins/cs4705-spring2019/hw1/h1_spring2019.pdf)

#### Check out the code on my [github](https://github.com/Sinyii/COMS4705-NLP/blob/master/viterbi/src/6.py).

Will upload more detail asap!