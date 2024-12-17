# **Week 01**

In another course, UX Design, I designed a housing search app based on LLM and RAG. Currently, we can recommend a variety of housing options based on user-provided information. However, we cannot yet predict housing price trends. Housing price trends are crucial, as they not only help tenants decide when to move in but also help landlords determine rental prices based on supply and demand.

Thus, my topic at this step is very clear: predicting rental prices over time for all neighborhoods in New York City.

For the dataset, I used publicly available data from **Streeteasy**. The dataset includes the median rent prices for all 155 neighborhoods in NYC after 2010, as well as the number of vacant rooms in these neighborhoods each month. The dataset cannot be directly used and contains some null values, so preprocessing and data merging are necessary. Splitting the data into training and testing sets is also meaningful. For evaluation, I will use traditional methods such as the mean square error (MSE) and mean absolute error (MAE) between predicted and actual results.

The question I want to explore is: **How can rental prices be predicted using move-in time, vacant room counts, and neighborhoods?**

Move-in time is a time series, but the number of vacant rooms and the neighborhood are not.

---

# **Week 02**

First, we need to preprocess the data. I downloaded two tables from Streeteasy's open data website, which record the number of vacant rooms and the median rent prices across different neighborhoods at various times.

**Download link**:  
[https://streeteasy.com/blog/download-data/](https://streeteasy.com/blog/download-data/)

- I obtained over **14,000 records** in Excel format, which I later converted into a DataFrame.
- After removing null values, I had around **10,000 records**.
- **Features**: `areaName`, `Year`, `Month`, `InventoryCount`  
- **Target variable**: `Price`

**Original data**:  

1.png

To process this data, we needed to make time a single variable and include it as a column in the table. The result looked like this:

![Processed Data Screenshot]  

Next, I merged the **price** and **inventory** tables, removing rows with null values:

![Merged Data Screenshot]  

Additionally, I split the time column into **Year** and **Month** features and encoded neighborhood names as numeric values because models can only accept numeric inputs.

![Final Preprocessed Data Screenshot]  

Like in our previous assignments, all variables were normalized at this stage. With this, the data was ready for modeling!

**Note**: Dear Professor Hersan, I did not save weekly versions of my Jupyter Notebook. However, you can find all the code in my final notebook. Some parts of the code might be commented out, but you will definitely find everything.

---

# **Week 03**

Since my data involves time series predictions, I chose to use an **LSTM model**. The reason is that RNN models tend to ignore long-term time dependencies when processing long sequences, leading to the vanishing gradient problem. LSTMs, on the other hand, have memory cells, and we can control the sequence length to ensure the model does not forget historical information for specific features.

### **Model Structure**

The model input parameters are divided into two parts:
1. **Sequence Features**: Features we want the model to retain over time.
2. **Static Features**: Features that do not change over time, such as `areaName` and `InventoryCount`.

For sequence data, I used **LSTM layers**, and for static features, I used **Dense layers**. The features were later fused at a higher level:

![Model Architecture Diagram]  

I generated the data structure for time series input and target values:

![Time Series Structure Screenshot]  

Here, `seq_length` is the **memory length** for the LSTM model. At each step, we use `seq_length` consecutive time points as input, and the target value is the value after the `seq_length` time points. To ensure uniform input size, we removed the first `seq_length` rows.

Finally, I constructed two input layers for **time series** and **static features**, connected them, and built relationships between inputs and outputs:

![Final Model Structure Screenshot]  

Using the prepared dataset, I trained the model with both feature types as inputs.

**Note**: Dear Professor Hersan, I did not save weekly versions of my Jupyter Notebook. However, you can find all the code in my final notebook.

---

# **Week 04**

After outputting results last week, I realized I forgot to **reverse the normalization**:

![Normalization Reversal Screenshot]  

After fixing this, I plotted the predictions and calculated the evaluation metrics. However, the results were not good:

![Poor Results Screenshot]  

### **Issues Identified**

1. **Issue 1**: Incorrect Month Encoding  
I used the same encoder for the month variable as for the neighborhood names, which is incorrect. Encoding months as arbitrary integers does not capture their sequential and cyclical nature. For example, "2019-01" encoded as 0 and "2019-02" encoded as 1 are just labels, not actual time relationships.

To address this, I used **sin and cos encoding** for months to reflect their cyclical nature:

![Sin-Cos Encoding Screenshot]  

2. **Issue 2**: Feature Division Problems  
In the LSTM input layer, I included both **month** and **year** since these are strongly time-dependent variables. Meanwhile, `areaName` and `InventoryCount` were treated as static features.

![Feature Division Screenshot]  

For the `Price` variable, I decided to use it as a **static input**. Although `Price` changes over time, treating it as a static input allows the model to reference short-term price values while focusing on overall trends.

---

### **Optimization Results**

To optimize the model, I wrapped all the code into a function with two variables:
1. **`seq_length`**: LSTM memory length.
2. **`epoch_size`**: Model iteration count.

I experimented with different combinations of `seq_length` and `epoch_size` and selected the model with the smallest error.

| seq_length | epoch_size | MSE                    | RMSE                  | MAE                   | R2                    |
| ---------- | ---------- | ---------------------- | --------------------- | --------------------- | --------------------- |
| 2          | 2          | 1221789.976940828      | 1105.3460892140652    | 766.2831308960443     | 0.23041283019969327   |
| 2          | 10         | 1824944.0861562665     | 1350.9049138100972    | 952.2712138662225     | -0.14950489103318843  |
| 2          | 20         | 1557648.2392418766     | 1248.057786819936     | 838.5944659689198     | 0.018860751351013172  |
| 2          | 30         | 4222.400250813167      | 64.97999885205576     | 43.178592509997216    | 0.9973403734519711    |
| 2          | 40         | 2219.0049882932744     | 47.10631580046644     | 24.92165901274549     | 0.998602282060793     |
| 2          | 50         | 42326.86130712687      | 205.73492972056755    | 141.43563958571357    | 0.9733389453059333    |
| 2          | 100        | 1275877.1301446394     | 1129.547312043475     | 1129.4014411051753    | 0.19634414413885415   |
| 2          | 150        | 2332.3732208686106     | 48.294650023254235    | 47.05361706880711     | 0.9985308731125291    |
| 2          | 200        | 35242.73367822462      | 187.73048148402705    | 187.13715952997623    | 0.9778011309804971    |
| 10         | 2          | 6355312.9226161195     | 2520.974597772877     | 2046.1505126406887    | -3.0013674568368733   |
| 10         | 10         | 4192023.4557723673     | 2047.4431508035498    | 1588.6565767956686    | -1.6393391542582885   |
| 10         | 20         | 5932955.014087635      | 2435.765796230753     | 2059.2526016398747    | -2.7354467679736283   |
| **10**     | **30**     | **1967.3052565219643** | **44.35431497072144** | **16.99454372744017** | **0.998761365281779** |
| 10         | 40         | 80104.31709202983      | 283.02706070626857    | 238.28642349001728    | 0.9495655349363589    |
| 10         | 50         | 31592.260297384124     | 177.74211739873058    | 50.48675713358045     | 0.9801092025237624    |
| 10         | 100        | 386881.4562974389      | 621.9979552196605     | 541.2422915953624     | 0.7564156340164941    |
| 10         | 150        | 150132.10394024113     | 387.4688425412308     | 300.634205069723      | 0.9054753523158315    |
| 10         | 200        | 19937244.660922825     | 4465.114182293979     | 4462.860033833448     | -11.552685121344423   |
| 20         | 2          | 234890743.96949655     | 15326.145763677721    | 13505.05487470187     | -146.8144728613517    |
| 20         | 10         | 55864788.89447125      | 7474.2751417425925    | 6692.726751207999     | -34.155171218748904   |
| 20         | 20         | 39421793.50655279      | 6278.6776877422835    | 5526.457340562587     | -23.807753289660834   |
| 20         | 30         | 3539352.156480954      | 1881.3166018724637    | 1439.266321192069     | -1.227280072597754    |
| 20         | 40         | 535334.9484007135      | 731.6658721033211     | 729.5401708119176     | 0.6631188957688923    |
| 20         | 50         | 6133536.138882598      | 2476.5976941931035    | 1857.0522519945862    | -2.859774956746351    |
| 20         | 100        | 12137844.247000523     | 3483.940907506975     | 3421.3656774145898    | -6.638227963876537    |
| 20         | 150        | 324896.2511082999      | 569.9967114890225     | 503.6776551900111     | 0.7955459322039558    |
| 20         | 200        | 2935.323724962932      | 54.17862793540394     | 8.377785552211213     | 0.9981528291760225    |
| 30         | 2          | 1333976114907.5544     | 1154978.8374284415    | 1154971.3873365042    | -839927.9405244429    |
| 30         | 10         | 155185733.91039667     | 12457.356618095055    | 11189.08881140516     | -96.71163637131582    |
| 30         | 20         | 10072780.235701323     | 3173.7643636069333    | 2417.922466287951     | -5.342257209076396    |
| 30         | 30         | 106708737.58784658     | 10329.992138808557    | 9134.270571633742     | -66.18842706795544    |
| 30         | 40         | 4936785.771535054      | 2221.8878845556214    | 1837.1541874097145    | -2.1084134088629773   |
| 30         | 50         | 2671978.0591134694     | 1634.6186280333004    | 1187.1276978119658    | -0.6823927169425095   |
| 30         | 100        | 44438052.35478102      | 6666.1872427033595    | 6640.946409536174     | -26.98011584780705    |
| 30         | 150        | 10109183.95857187      | 3179.494292898144     | 2512.753432260942     | -5.365178564294156    |
| 30         | 200        | 56457034.76584063      | 7513.789640776526     | 6590.551496415397     | -34.54778595065804    |

At `epoch_size = 30` and `seq_length = 10`, the model achieved its best performance, with an average price prediction error of **$17**.

![Best Model Results Screenshot]  

---

### **Future Work**

I noticed that the model's accuracy decreases for higher-priced properties. This is likely due to insufficient data for this price range. To further optimize the model, more data for high-priced properties needs to be collected.

**Note**: Dear Professor Hersan, I did not save weekly versions of my Jupyter Notebook. However, you can find all the code in my final notebook.

--- 

Let me know if you need further refinements or additional formatting! 😊
