# Time-Series Anomaly Detection in Network Traffic

## Overview

This project utilizes the CIC-IDS2017 dataset from the Canadian Institute for Cybersecurity, which includes real-world network traffic captures of both benign activity and various attacks like DDoS and Brute Force. To enable deep learning-based detection, we preprocess the raw flow data into time-series sequences using a sliding window technique. By focusing on temporal features such as flow duration and packet inter-arrival times, we train LSTM models to recognize "normal" network rhythms and identify anomalous spikes or patterns that signify potential intrusions.

- **Dataset Access:** https://www.unb.ca/cic/datasets/ids-2017.html
