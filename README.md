# 🔍 IEEE-CIS Fraud Detection

## კონკურსის მიმოხილვა
Kaggle-ის კონკურსი "IEEE-CIS Fraud Detection" გულისხმობს საბანკო ტრანზაქციების თაღლითობის აღმოჩენას. მონაცემთა ნაკრები შეიცავს ტრანზაქციულ და იდენტიფიკაციის მონაცემებს, სულ 590,000+ ჩანაწერით და 400+ მახასიათებლით. შეფასება ხდება ROC-AUC მეტრიკით.

## პრობლემის გადაჭრის მიდგომა
მონაცემების გაერთიანება → გაწმენდა → Feature Engineering → Feature Selection → მოდელების ტრენინგი → საუკეთესო მოდელის რეგისტრაცია Model Registry-ში → Kaggle-ზე გაგზავნა. ყველა ექსპერიმენტი დალოგილია MLflow-ზე DagsHub-ის გამოყენებით.

## რეპოზიტორიის სტრუქტურა
IEEE-Fraud-Detection/

├── model_experiment_RandomForest.ipynb   # Random Forest-ის ექსპერიმენტები 

├── model_experiment_XGBoost.ipynb        # XGBoost-ის ექსპერიმენტები

├── model_inference.ipynb                 # პროგნოზირება და submission

└── README.md                             # პროექტის დოკუმენტაცია

## ფაილების განმარტება
- **model_experiment_RandomForest.ipynb** - Random Forest მოდელის სრული pipeline: cleaning, feature engineering, feature selection და 4 სხვადასხვა კონფიგურაციის ტრენინგი MLflow logging-ით
- **model_experiment_XGBoost.ipynb** - XGBoost მოდელის სრული pipeline: cleaning, feature engineering, feature selection და 2 სხვადასხვა კონფიგურაციის ტრენინგი MLflow logging-ით
- **model_inference.ipynb** - საუკეთესო მოდელის ჩამოტვირთვა MLflow Model Registry-დან და submission.csv-ის გენერაცია
- **README.md** - პროექტის სრული დოკუმენტაცია

## Feature Engineering

### მონაცემების გაერთიანება
- **train_transaction + train_identity** და **test_transaction + test_identity** გაერთიანებული იქნა TransactionID-ის მიხედვით left join-ით
- გაერთიანების შემდეგ train shape: (590,540 × 432), test shape: (506,691 × 432)

### კატეგორიული ცვლადების დამუშავება
- **Label Encoding** - გამოყენებული იქნა 26 კატეგორიული სვეტისთვის (M1-M9, ProductCD, card4, card6 და სხვა)
- Encoding-ისთვის train და test მონაცემები გაერთიანდა, რომ უცნობი კატეგორიები სატესტო სეტში არ გამოიწვიოს შეცდომა

### ახალი მახასიათებლების შექმნა
- **Transaction_hour** - ტრანზაქციის საათი (TransactionDT / 3600 % 24)
- **Transaction_day** - კვირის დღე (TransactionDT / 86400 % 7)
- **Transaction_week** - კვირის ნომერი (TransactionDT / 604800)
- **TransactionAmt_log** - ტრანზაქციის თანხის ლოგარითმული გარდაქმნა (მარჯვნივ დახრილი განაწილების გამოსასწორებლად)
- **TransactionAmt_cents** - ტრანზაქციის თანხის ცენტების ნაწილი (თაღლითობის პატერნების დასაფიქსირებლად)

Feature Engineering-ის შემდეგ მახასიათებლების რაოდენობა გაიზარდა 417-დან 422-მდე.

### NaN მნიშვნელობების დამუშავება
- რიცხვითი სვეტები შეივსო -999-ით Feature Selection-ამდე (Random Forest-ი NaN-ს ვერ ამუშავებს)
- კატეგორიული სვეტები Label Encoding-მდე გარდაიქმნა სტრინგად

### Cleaning მიდგომები
- **>90% missing სვეტები** - წაიშალა 12 სვეტი სადაც ცარიელი მნიშვნელობები 90%-ს აღემატებოდა
- **მაღალი კარდინალობის სვეტები** - წაიშალა 3 კატეგორიული სვეტი 100+ უნიკალური მნიშვნელობით
- **მეხსიერების ოპტიმიზაცია** - რიცხვითი სვეტები გარდაიქმნა მინიმალური სიზუსტის ტიპებში (int8, int16, float32) მეხსიერების დასაზოგად
- გაწმენდის შემდეგ საბოლოო shape: (590,540 × 417)

## Feature Selection

### Random Forest-ისთვის
გამოყენებული იქნა **Random Forest Feature Importance** მიდგომა:
- სწრაფი Random Forest (100 ხე, max_depth=8) მოვარჯიშეთ სრულ მონაცემებზე
- შევინახეთ მხოლოდ ის მახასიათებლები რომელთა importance >= 0.0001
- შედეგად შეირჩა **369 მახასიათებელი** 422-დან (53 ამოღებულ იქნა)

### XGBoost-ისთვის
- მაღალი missing rate-ის სვეტები (>90%) ამოღებულ იქნა
- მაღალი კარდინალობის სვეტები (>500 უნიკალური მნიშვნელობა) ამოღებულ იქნა
- Feature Engineering-ის შემდეგ დარჩა 424 მახასიათებელი

## Training

### Random Forest - გატესტებული მოდელები

| ვერსია | Val AUC | Train AUC | Gap | შენიშვნა |
|--------|---------|-----------|-----|---------|
| v1_underfitted | 0.84999 | - | - | ✗ Underfitted |
| v2_overfitted | 0.94299 | 1.00000 | 0.057 | ✗ Overfitted |
| v3_balanced | 0.91436 | 0.94911 | 0.035 | ✅ საუკეთესო |
| v4_strict_reg | 0.91272 | 0.93713 | 0.024 | კარგი, მაგრამ v3-ზე სუსტი |

### XGBoost - გატესტებული მოდელები

| ვერსია | CV AUC | შენიშვნა |
|--------|--------|---------|
| v1_baseline | **0.93226** | ✅ საუკეთესო |
| v2_tuned | 0.90924 | ✗ Underfitted |

### Underfitting და Overfitting ანალიზი

**Random Forest v1 - Underfitting:**
- max_depth=5 და min_samples_leaf=50 ძალიან შემზღუდველი პარამეტრებია 590,000+ სტრიქონიანი მონაცემებისთვის
- მოდელი ვერ ასახავდა მონაცემების სტრუქტურას, Val AUC მხოლოდ 0.850
- **გამომწვევი მიზეზი:** ხეები ძალიან არაღრმა იყო და თითოეულ ფოთოლს ჰქონდა მინიმუმ 50 ობიექტი, რაც ზედმეტ გამარტივებას იწვევდა

**Random Forest v2 - Overfitting:**
- max_depth=None (შეუზღუდავი სიღრმე) და min_samples_leaf=1 იწვევს ზეპირ დამახსოვრებას
- Train AUC = 1.00000 ყველა fold-ში, Val AUC = 0.943, Gap = 0.057
- მიუხედავად იმისა რომ Val AUC უმაღლესია, მოდელი სარეალიზაციო მონაცემებზე სავარაუდოდ უარესად იმუშავებს
- **გამომწვევი მიზეზი:** შეუზღუდავი სიღრმის ხეები train მონაცემებს ზეპირად სწავლობენ

**XGBoost v2 - Underfitting:**
- reg_alpha=0.1, reg_lambda=1.5 და learning_rate=0.03 კომბინაციამ გადაჭარბებული რეგულარიზაცია გამოიწვია
- CV AUC დაეცა 0.932-დან 0.909-მდე
- **გამომწვევი მიზეზი:** ძალიან ძლიერი რეგულარიზაცია კომბინაციაში დაბალ learning_rate-თან

**საუკეთესო ბალანსი:**
- Random Forest v3: Train/Val gap = 0.035, სტაბილური შედეგები (std: ±0.00095)
- XGBoost v1: CV AUC = 0.932, სტაბილური ყველა fold-ში (std: ±0.00202)

### Hyperparameter ოპტიმიზაცია

**Random Forest v3 (საუკეთესო):**
- n_estimators: 300, max_depth: 15, min_samples_leaf: 10, max_features: sqrt

**Random Forest v4 (strict regularization):**
- n_estimators: 500, max_depth: 12, min_samples_leaf: 20, min_samples_split: 10, max_features: 0.5, max_samples: 0.8

**XGBoost v1 (საუკეთესო):**
- n_estimators: 300, max_depth: 6, learning_rate: 0.1, subsample: 0.8, colsample_bytree: 0.8

**XGBoost v2 (tuned):**
- n_estimators: 500, max_depth: 4, learning_rate: 0.03, reg_alpha: 0.1, reg_lambda: 1.5

### საბოლოო მოდელის შერჩევის დასაბუთება
XGBoost v1 შეირჩა საბოლოო მოდელად რადგან:
1. უმაღლესი CV AUC ყველა მოდელს შორის (0.93226)
2. სტაბილური შედეგები ყველა fold-ში (std: ±0.00202)
3. Random Forest v3-თან შედარებით (0.914) მნიშვნელოვნად უკეთესი შედეგი
4. Random Forest v2 (0.943) უარყოფილ იქნა overfitting-ის გამო (Train AUC = 1.0, Gap = 0.057)

## MLflow Tracking

### ექსპერიმენტების ბმული
🔗 [DagsHub MLflow Experiments](https://dagshub.com/mgior23/IEEE-Fraud-Detection.mlflow)

### MLflow სტრუქტურა

**RandomForest_Training ექსპერიმენტი:**
- `RandomForest_Cleaning` - გაწმენდის ეტაპის პარამეტრები და მეტრიკები
- `RandomForest_FeatureEngineering` - feature engineering-ის ინფო
- `RandomForest_FeatureSelection` - შერჩეული მახასიათებლები და importance გრაფიკი
- `RandomForest_v1_underfitted` - განზრახ underfitted მოდელი
- `RandomForest_v2_overfitted` - განზრახ overfitted მოდელი
- `RandomForest_v3_balanced` - დაბალანსებული მოდელი
- `RandomForest_v4_strict_reg` - მკაცრი რეგულარიზაციის მოდელი
- `RandomForest_Final_Model` - საბოლოო მოდელი რეგისტრაციით

**XGBoost_Training ექსპერიმენტი:**
- `XGBoost_Cleaning` - გაწმენდის ეტაპის ლოგი
- `XGBoost_Feature_Engineering` - feature engineering-ის ლოგი
- `XGBoost_v1_baseline` - baseline მოდელი
- `XGBoost_v2_tuned` - tuned მოდელი
- `XGBoost_Final_Model` - საბოლოო მოდელი რეგისტრაციით

### ჩაწერილი მეტრიკები
- **cv_auc_mean** - 5-Fold Cross Validation საშუალო AUC
- **cv_auc_std** - Cross Validation სტანდარტული გადახრა
- **train_auc_mean** - სასწავლო სეტზე საშუალო AUC (overfitting-ის შესაფასებლად)
- **overfit_gap** - Train და Val AUC-ს სხვაობა
- **n_features** - გამოყენებული მახასიათებლების რაოდენობა

### საუკეთესო მოდელის შედეგები
- **მოდელი:** XGBoost v1 (baseline params)
- **CV AUC:** 0.93226 (±0.00202)
- **Kaggle Fraud Rate:** 2.02%
- **Model Registry:** XGBoost_FraudDetection v1