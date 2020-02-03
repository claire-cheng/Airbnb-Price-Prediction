## **Project Aims and Background**
Airbnb is a home-sharing internet platform that allows home-owners to market their properties online for short-term rentals. The renters, also called the hosts, are responsible for setting the prices on their own listings based on their intuition. Without a proper guidance on the pricing criterias, it could be very challenging for the hosts not to overprice their listings and at the same time selling short on their homes. Therefore, this project acts as the middle man by fitting a statistical model with several listing features to predict optimal prices of the listings for the hosts.


## **Data Exploration**
Two different sets of datasets were given, one was the analysis data and the other one was the scoring data, which was used for our submissions to Kaggle. The only difference between the two datasets were the number of observations and the additional Price variable in the analysis dataset. I started my data exploration process with the analysis dataset. This dataset contained 91 variables including Price, the outcome. It was challenging to manipulate the data without fully understood the variables, soI started eyeballing and exploring all the 90 variables one by one following the steps listed below:
 - Removed variables that were irrelevant to Price.   
 - Removed variables with high ratios of NAs.
 - Removed variables that contained only one value or one level.
 - Removed any variables that held redundant information.

## **Data Transformation**
After removing some variables through the manual process, I then conducted data transformation on some of the variables. 
1. Identified NAs and performed the following transformations on some variables.
   - Imputed NA values using the Caret package for security_deposit (35% were NAs) and cleaning_fee (16% were NAs). 
     ```markdown
     library(caret)
     analysis_caret <- predict(preProcess(analysisData_clean,method='bagImpute'),newdata=analysisData_clean)
     analysisData_clean$security_deposit <- round(analysis_caret$security_deposit)
     analysisData_clean$cleaning_fee <- round(analysis_caret$cleaning_fee)
     ```
   - There were some 0s in Price, so I thought I would set those 0s to NAs and impute them with the Caret package in order to get better predictions.
     ```markdown
     analysisData_clean$price[analysisData_clean$price==0] = NA
     analysisData_clean$price <- round(analysis_caret$price)
     ```
   - Converted NAs in host_about to empty strings for future transformation.
     ```markdown
     analysisData_clean$host_about[is.na(analysisData_clean$host_about)] <- ''
     ```
   - I also removed 3 rows with number_of_reviews = 0 because there were few variables associated with reviews. With number_of_reviews being 0, all of the other variables were also NAs.
     ```markdown
     analysisData_clean <- subset(analysisData_clean, number_of_reviews > 0)
     ```
   - I then used rowSums to identify all the remaining NAs in the dataset and removed them. There were 15 of them left after performing the previous steps.
     ```markdown
     analysisData_clean <- subset(analysisData_clean, rowSums(is.na(analysisData_clean))==0)
     ```
2. Converted characters to numeric values.
   - Some of the variables were factors with only two levels “t” or “f”, so I converted them into 1 for “t”s and 0 for “f”s and empty strings for the following variables:
      - Host_is_superhost
      - Host_has_profile_pic
      - Host_identity_verified
      - Require_guest_profile_picture
      ```markdown
      analysisData_clean$host_is_superhost <- as.numeric(ifelse(analysisData_clean$host_is_superhost == 't', 1, 0))
      analysisData_clean$host_has_profile_pic <- as.numeric(ifelse(analysisData_clean$host_has_profile_pic == 't', 1, 0))
      analysisData_clean$host_identity_verified <- as.numeric(ifelse(analysisData_clean$host_identity_verified == 't', 1, 0))
      analysisData_clean$price <- as.numeric(analysisData_clean$price)
      analysisData_clean$extra_people <- as.numeric(analysisData_clean$extra_people)
      analysisData_clean$require_guest_profile_picture <- as.numeric(ifelse(analysisData_clean$require_guest_profile_picture == 't', 1, 0))
      ```
3. Converted class from integers to numeric.
   - Some of the variables were more suitable as numerics other than integers, so I converted the following variables:
      - Price
      - Extra_people
      - Require_guest_profile_picture
      ```markdown
      analysisData_clean$price <- as.numeric(analysisData_clean$price)
      analysisData_clean$extra_people <- as.numeric(analysisData_clean$extra_people)
      analysisData_clean$require_guest_profile_picture <- as.numeric(ifelse(analysisData_clean$require_guest_profile_picture == 't', 1, 0))
      ```
4. Transformed existing variables into new calculated variables.
   - Since the date fields were not meaningful in fitting models, I used them to calculate the length of hosting and length of listing in days and created new variables to store them.
      ```markdown
     analysisData_clean <- analysisData_clean %>% 
     mutate(listing_duration = as.numeric(ymd(analysisData_clean$last_review)-ymd(analysisData_clean$first_review)),
     hosting_duration = as.numeric(ymd(analysisData_clean$last_review)-ymd(analysisData_clean$host_since)))
      ```
   - The existing variable, host_local stored information of where the host was located. Since the hosts could be located in any location in the world, I chose to only look at the ones that were in New York, locally, and set them to 1, anywhere else 0. 
      ```markdown
     analysisData_clean <- analysisData_clean %>% 
     mutate(host_local = as.numeric(str_detect(host_location, 'New York|new york|NY')))
      ```   
   - Prices could be affected for those hosts who provided some personal information, so I calculated the length of host_about and stored it in a new variable- host_about_len.
      ```markdown
     analysisData_clean <- analysisData_clean %>% 
     mutate(host_about_len = ifelse(is.na(host_about), 0, str_length(host_about)))
      ```
   - Amenities contained all the amenities the Airbnb provided into a string separated by “,”. I decided to clean the variable, amenities, by removing all the spaces and separated all the amenities by “,”. After storing unique amenities into a list, I created a new variable to count the total number of amenities there are on the listing.
      ```markdown
     analysisData_clean$amenities <- gsub(" ", '', analysisData_clean$amenities, fixed = T)
     analysisData_clean <- analysisData_clean %>% 
     mutate(total_amenities = ifelse(str_length(amenities)>2, str_count(amenities, ',')+1, 0))
      ```
   - In order to take all the amenities into consideration without using NLP, I created variables for each of the amenities and assigned a 1 if that amenity was included in the unit and 0 if not.
      ```markdown
     analysisData_clean$WIFI <- as.numeric(str_detect(analysisData_clean$amenities, 'Wifi'))
     analysisData_clean$TV <- as.numeric(str_detect(analysisData_clean$amenities, 'TV|tv|Tv'))
     analysisData_clean$AC <- as.numeric(str_detect(analysisData_clean$amenities, 'Airconditioning'))
     analysisData_clean$Kitchen <- as.numeric(str_detect(analysisData_clean$amenities, 'Kitchen'))
     analysisData_clean$Washer <- as.numeric(str_detect(analysisData_clean$amenities, 'Washer'))
     analysisData_clean$Dryer <- as.numeric(str_detect(analysisData_clean$amenities, 'Dryer'))
     analysisData_clean$Hairdryer <- as.numeric(str_detect(analysisData_clean$amenities, 'Hairdryer'))
     analysisData_clean$Heating <- as.numeric(str_detect(analysisData_clean$amenities, 'Heating'))
     analysisData_clean$Elevator <- as.numeric(str_detect(analysisData_clean$amenities, 'Elevator'))
     analysisData_clean$Freeparking <- as.numeric(str_detect(analysisData_clean$amenities, 'Freestreetparking'))
     analysisData_clean$Petsallowed <- as.numeric(str_detect(analysisData_clean$amenities, 'Petsallowed'))
     analysisData_clean$Gym <- as.numeric(str_detect(analysisData_clean$amenities, 'Gym'))
     analysisData_clean$Essentials <- as.numeric(str_detect(analysisData_clean$amenities, 'Essentials'))
     analysisData_clean$Iron <- as.numeric(str_detect(analysisData_clean$amenities, 'Iron'))
     analysisData_clean$Privatelivingroom <- as.numeric(str_detect(analysisData_clean$amenities, 'Privatelivingroom'))
     analysisData_clean$Indoorfireplace <- as.numeric(str_detect(analysisData_clean$amenities, 'Indoorfireplace'))
     analysisData_clean$Microwave <- as.numeric(str_detect(analysisData_clean$amenities, 'Microwave'))
     analysisData_clean$Hotwater <- as.numeric(str_detect(analysisData_clean$amenities, 'Hotwater'))
     analysisData_clean$Hangers <- as.numeric(str_detect(analysisData_clean$amenities, 'Hangers'))
     analysisData_clean$Bedlinens <- as.numeric(str_detect(analysisData_clean$amenities, 'Bedlinens'))
     analysisData_clean$Extrapillowsandblankets <- as.numeric(str_detect(analysisData_clean$amenities, 'Extrapillowsandblankets'))
     analysisData_clean$Coffeemaker <- as.numeric(str_detect(analysisData_clean$amenities, 'Coffeemaker'))
     analysisData_clean$Refrigerator <- as.numeric(str_detect(analysisData_clean$amenities, 'Refrigerator'))
     analysisData_clean$Dishesandsilverware <- as.numeric(str_detect(analysisData_clean$amenities, 'Dishesandsilverware'))
     analysisData_clean$Cookingbasics <- as.numeric(str_detect(analysisData_clean$amenities, 'Cookingbasics'))
     analysisData_clean$Luggagedropoffallowed <- as.numeric(str_detect(analysisData_clean$amenities, 'Luggagedropoffallowed'))
     analysisData_clean$Familykidfriendly <- as.numeric(str_detect(analysisData_clean$amenities, 'Family/kidfriendly'))
     analysisData_clean$Internet <- as.numeric(str_detect(analysisData_clean$amenities, 'Internet'))
     analysisData_clean$Doorman <- as.numeric(str_detect(analysisData_clean$amenities, 'Doorman'))
     analysisData_clean$alldaycheckin <- as.numeric(str_detect(analysisData_clean$amenities, '24-hourcheck-in'))
     analysisData_clean$Privatebathroom <- as.numeric(str_detect(analysisData_clean$amenities, 'Privatebathroom'))
     analysisData_clean$Buzzerwirelessintercom <- as.numeric(str_detect(analysisData_clean$amenities, 'Buzzer/wirelessintercom'))
     analysisData_clean$Safetycard <- as.numeric(str_detect(analysisData_clean$amenities, 'Safetycard'))
     analysisData_clean$Paidparkingoffpremises <- as.numeric(str_detect(analysisData_clean$amenities, 'Paidparkingoffpremises'))
     analysisData_clean$Selfcheckin <- as.numeric(str_detect(analysisData_clean$amenities, 'Selfcheck-in'))
     analysisData_clean$Smartlock <- as.numeric(str_detect(analysisData_clean$amenities, 'Smartlock'))
     analysisData_clean$Bathtub <- as.numeric(str_detect(analysisData_clean$amenities, 'Bathtub'))
     analysisData_clean$Dishwasher <- as.numeric(str_detect(analysisData_clean$amenities, 'Dishwasher'))
     analysisData_clean$Oven <- as.numeric(str_detect(analysisData_clean$amenities, 'Oven'))
     analysisData_clean$Stove <- as.numeric(str_detect(analysisData_clean$amenities, 'Stove'))
     analysisData_clean$Patioorbalcony <- as.numeric(str_detect(analysisData_clean$amenities, 'Patioorbalcony'))
     analysisData_clean$Longtermstaysallowed <- as.numeric(str_detect(analysisData_clean$amenities, 'Longtermstaysallowed'))
     analysisData_clean$Widehallwayclearance <- as.numeric(str_detect(analysisData_clean$amenities, 'Widehallwayclearance'))
     analysisData_clean$Wideclearancetobed <- as.numeric(str_detect(analysisData_clean$amenities, 'Wideclearancetobed'))
     analysisData_clean$Wideclearancetoshower <- as.numeric(str_detect(analysisData_clean$amenities, 'Wideclearancetoshower'))
     analysisData_clean$toilet <- as.numeric(str_detect(analysisData_clean$amenities, 'toilet'))
     analysisData_clean$Paidparkingonpremises <- as.numeric(str_detect(analysisData_clean$amenities, 'Paidparkingonpremises'))
     analysisData_clean$Petsliveonthisproperty <- as.numeric(str_detect(analysisData_clean$amenities, 'Petsliveonthisproperty'))
     analysisData_clean$Dogs <- as.numeric(str_detect(analysisData_clean$amenities, 'Dog(s)'))
     analysisData_clean$Lockbox <- as.numeric(str_detect(analysisData_clean$amenities, 'Lockbox'))
     analysisData_clean$Singlelevelhome <- as.numeric(str_detect(analysisData_clean$amenities, 'Singlelevelhome'))
     analysisData_clean$Cleaningbeforecheckout <- as.numeric(str_detect(analysisData_clean$amenities, 'Cleaningbeforecheckout'))
     analysisData_clean$Widedoorway <- as.numeric(str_detect(analysisData_clean$amenities, 'Widedoorway'))
     analysisData_clean$Accessibleheightbed <- as.numeric(str_detect(analysisData_clean$amenities, 'Accessible-heightbed'))
     analysisData_clean$Stepfreeaccess <- as.numeric(str_detect(analysisData_clean$amenities, 'Step-freeaccess'))
     analysisData_clean$Accessibleheighttoilet <- as.numeric(str_detect(analysisData_clean$amenities, 'Accessible-heighttoilet'))
     analysisData_clean$Wideentryway <- as.numeric(str_detect(analysisData_clean$amenities, 'Wideentryway'))
     analysisData_clean$Windowguards <- as.numeric(str_detect(analysisData_clean$amenities, 'Windowguards'))
     analysisData_clean$Breakfast <- as.numeric(str_detect(analysisData_clean$amenities, 'Breakfast'))
     analysisData_clean$Gardenorbackyard <- as.numeric(str_detect(analysisData_clean$amenities, 'Gardenorbackyard'))
     analysisData_clean$BBQgrill <- as.numeric(str_detect(analysisData_clean$amenities, 'BBQgrill'))
     analysisData_clean$Smokingallowed <- as.numeric(str_detect(analysisData_clean$amenities, 'Smokingallowed'))
     analysisData_clean$Suitableforevents <- as.numeric(str_detect(analysisData_clean$amenities, 'Suitableforevents'))
     analysisData_clean$Buildingstaff <- as.numeric(str_detect(analysisData_clean$amenities, 'Buildingstaff'))
     analysisData_clean$Freeparkingonpremises <- as.numeric(str_detect(analysisData_clean$amenities, 'Freeparkingonpremises'))
     analysisData_clean$Roomdarkeningshades <- as.numeric(str_detect(analysisData_clean$amenities, 'Room-darkeningshades'))
     analysisData_clean$Handheldshowerhead <- as.numeric(str_detect(analysisData_clean$amenities, 'Handheldshowerhead'))
     analysisData_clean$Keypad <- as.numeric(str_detect(analysisData_clean$amenities, 'Keypad'))
     analysisData_clean$Wheelchairaccessible <- as.numeric(str_detect(analysisData_clean$amenities, 'Wheelchairaccessible'))
     analysisData_clean$Childrensbooksandtoys <- as.numeric(str_detect(analysisData_clean$amenities, 'Children’sbooksandtoys'))
     analysisData_clean$Crib <- as.numeric(str_detect(analysisData_clean$amenities, 'Crib'))
     analysisData_clean$Cats <- as.numeric(str_detect(analysisData_clean$amenities, 'Cat(s)'))
     analysisData_clean$Highchair <- as.numeric(str_detect(analysisData_clean$amenities, 'Highchair'))
     analysisData_clean$Flatpathtofrontdoor <- as.numeric(str_detect(analysisData_clean$amenities, 'Flatpathtofrontdoor'))
     analysisData_clean$Pocketwifi <- as.numeric(str_detect(analysisData_clean$amenities, 'Pocketwifi'))
     analysisData_clean$Hottub <- as.numeric(str_detect(analysisData_clean$amenities, 'Hottub'))
     analysisData_clean$Waterfront <- as.numeric(str_detect(analysisData_clean$amenities, 'Waterfront'))
     analysisData_clean$Pool <- as.numeric(str_detect(analysisData_clean$amenities, 'Pool'))
     analysisData_clean$Babybath <- as.numeric(str_detect(analysisData_clean$amenities, 'Babybath'))
     analysisData_clean$Widehallways <- as.numeric(str_detect(analysisData_clean$amenities, 'Widehallways'))
     analysisData_clean$Nostairsorstepstoenter <- as.numeric(str_detect(analysisData_clean$amenities, 'Nostairsorstepstoenter'))
     analysisData_clean$Extraspacearoundbed <- as.numeric(str_detect(analysisData_clean$amenities, 'Extraspacearoundbed'))
     analysisData_clean$Wideentrance <- as.numeric(str_detect(analysisData_clean$amenities, 'Wideentrance'))
     analysisData_clean$Fullkitchen <- as.numeric(str_detect(analysisData_clean$amenities, 'Fullkitchen'))
     analysisData_clean$Fixedgrabbarsforshower <- as.numeric(str_detect(analysisData_clean$amenities, 'Fixedgrabbarsforshower'))
     analysisData_clean$Gameconsole <- as.numeric(str_detect(analysisData_clean$amenities, 'Gameconsole'))
     analysisData_clean$Beachessentials <- as.numeric(str_detect(analysisData_clean$amenities, 'Beachessentials'))
     analysisData_clean$Fireplaceguards <- as.numeric(str_detect(analysisData_clean$amenities, 'Fireplaceguards'))
     analysisData_clean$Changingtable <- as.numeric(str_detect(analysisData_clean$amenities, 'Changingtable'))
     analysisData_clean$Fixedgrabbarsfortoilet <- as.numeric(str_detect(analysisData_clean$amenities, 'Fixedgrabbarsfortoilet'))
     analysisData_clean$v <- as.numeric(str_detect(analysisData_clean$amenities, 'EVcharger'))
     analysisData_clean$Otherpet <- as.numeric(str_detect(analysisData_clean$amenities, 'Otherpet(s)'))
     analysisData_clean$Disabledparkingspot <- as.numeric(str_detect(analysisData_clean$amenities, 'Disabledparkingspot'))
     analysisData_clean$Widedoorwaytoguestbathroom <- as.numeric(str_detect(analysisData_clean$amenities,      'Widedoorwaytoguestbathroom'))
     analysisData_clean$Beachfront <- as.numeric(str_detect(analysisData_clean$amenities, 'Beachfront'))
     analysisData_clean$Firmmattress <- as.numeric(str_detect(analysisData_clean$amenities, 'Firmmattress'))
     analysisData_clean$Showerchair <- as.numeric(str_detect(analysisData_clean$amenities, 'Showerchair'))
     analysisData_clean$Mobilehoist <- as.numeric(str_detect(analysisData_clean$amenities, 'Mobilehoist'))
     analysisData_clean$Groundflooraccess <- as.numeric(str_detect(analysisData_clean$amenities, 'Groundflooraccess'))
     analysisData_clean$Lakeaccess <- as.numeric(str_detect(analysisData_clean$amenities, 'Lakeaccess'))
     analysisData_clean$Babymonitor <- as.numeric(str_detect(analysisData_clean$amenities, 'Babymonitor'))
     analysisData_clean$Hotwaterkettle <- as.numeric(str_detect(analysisData_clean$amenities, 'Hotwaterkettle'))
     analysisData_clean$Electricprofilingbed <- as.numeric(str_detect(analysisData_clean$amenities, 'Electricprofilingbed'))
     analysisData_clean$Airpurifier <- as.numeric(str_detect(analysisData_clean$amenities, 'Airpurifier'))
     analysisData_clean$SkiinSkiout <- as.numeric(str_detect(analysisData_clean$amenities, 'Ski-in/Ski-out'))
     analysisData_clean$Extraspacearoundshowerandtoilet <- as.numeric(str_detect(analysisData_clean$amenities, 'Extraspacearoundshowerandtoilet'))
     analysisData_clean$Kitchenette <- as.numeric(str_detect(analysisData_clean$amenities, 'Kitchenette'))
     analysisData_clean$Poolwithpoolhoist <- as.numeric(str_detect(analysisData_clean$amenities, 'Poolwithpoolhoist'))
     analysisData_clean$Ceilinghoist <- as.numeric(str_detect(analysisData_clean$amenities, 'Ceilinghoist'))
     analysisData_clean$Smokedetector <- as.numeric(str_detect(analysisData_clean$amenities, 'Smokedetector'))
     analysisData_clean$Carbonmonoxidedetector <- as.numeric(str_detect(analysisData_clean$amenities, 'Carbonmonoxidedetector'))
     analysisData_clean$Laptopfriendlyworkspace <- as.numeric(str_detect(analysisData_clean$amenities, 'Laptopfriendlyworkspace'))
     analysisData_clean$Privateentrance <- as.numeric(str_detect(analysisData_clean$amenities, 'Privateentrance'))
     analysisData_clean$Shampoo <- as.numeric(str_detect(analysisData_clean$amenities, 'Shampoo'))
     analysisData_clean$Lockonbedroomdoor <- as.numeric(str_detect(analysisData_clean$amenities, 'Lockonbedroomdoor'))
     analysisData_clean$Hostgreetsyou <- as.numeric(str_detect(analysisData_clean$amenities, 'Hostgreetsyou'))
     analysisData_clean$Firstaidkit <- as.numeric(str_detect(analysisData_clean$amenities, 'Firstaidkit'))
     analysisData_clean$Fireextinguisher <- as.numeric(str_detect(analysisData_clean$amenities, 'Fireextinguisher'))
      ```
   - Dummy variables- I didn’t want to include any categorical variables in my model, so I created dummy variables for all categorical variables and assigned them numeric values.
     - I created dummy variables for each level of the variables listed below and assigned a 1 if that level was presented in the row, if not, 0.
        -  Neighbourhood_group_cleansed
        -  Property_type
        -  Room_type
        -  Bed_type
        -  Cancellation_policy
        -  Neighbourhood_cleansed
      ```markdown
      analysisData_clean$neighbourhood_group_cleansed <- str_replace_all(analysisData_clean$neighbourhood_group_cleansed, "[^[:alnum:]]", "_")
      analysisData_clean$property_type <- str_replace_all(analysisData_clean$property_type, "[^[:alnum:]]", "_")
      analysisData_clean$room_type <- str_replace_all(analysisData_clean$room_type, "[^[:alnum:]]", "_")
      analysisData_clean$bed_type <- str_replace_all(analysisData_clean$bed_type, "[^[:alnum:]]", "_")
      analysisData_clean$neighbourhood_cleansed <- str_replace_all(analysisData_clean$neighbourhood_cleansed, "[^[:alnum:]]", "_")
      analysisData_clean = dummy_cols(analysisData_clean,select_columns = "neighbourhood_group_cleansed")
      analysisData_clean = dummy_cols(analysisData_clean,select_columns = "property_type")
      analysisData_clean = dummy_cols(analysisData_clean,select_columns = "room_type")
      analysisData_clean = dummy_cols(analysisData_clean,select_columns = "bed_type")
      analysisData_clean = dummy_cols(analysisData_clean,select_columns = "cancellation_policy")
      analysisData_clean = dummy_cols(analysisData_clean,select_columns = "neighbourhood_cleansed")
      ```
5. Reduced the gaps between values
   - The gaps between the prices of security_deposit were quite significant, so I decided to group the prices into a smaller scale of 0 to 10.
    ```markdown
     analysisData_clean_nzv$security_deposit <- ifelse(analysisData_clean_nzv$security_deposit ==     0,0,ifelse(analysisData_clean_nzv$security_deposit>0 & analysisData_clean_nzv$security_deposit <=100,1,ifelse(analysisData_clean_nzv$security_deposit>100 &         analysisData_clean_nzv$security_deposit<=200,2,ifelse(analysisData_clean_nzv$security_deposit>200 & analysisData_clean_nzv$security_deposit<=300,3,ifelse(analysisData_clean_nzv$security_deposit>300 & analysisData_clean_nzv$security_deposit<=400,4,ifelse(analysisData_clean_nzv$security_deposit>400 & analysisData_clean_nzv$security_deposit<=500,5,ifelse(analysisData_clean_nzv$security_deposit>500 & analysisData_clean_nzv$security_deposit<=600,6,ifelse(analysisData_clean_nzv$security_deposit>600 & analysisData_clean_nzv$security_deposit<=700,7,ifelse(analysisData_clean_nzv$security_deposit>700 & analysisData_clean_nzv$security_deposit<=800,8,ifelse(analysisData_clean_nzv$security_deposit>800 & analysisData_clean_nzv$security_deposit<=900,9,ifelse(analysisData_clean_nzv$security_deposit>900 & analysisData_clean_nzv$security_deposit<=1000,10,NA)))))))))))
     ```
    
## **Data Splitting**
In order to predict the price of a listing, I used the process in the caret package to split data into train and test sets. Of which, train dataset consisted of 70% of the original data and 30% were split to test.
    ```{r}
    library(caret)
    set.seed(61710)
    split = createDataPartition(y=analysisData_clean_nzv$price,p=0.7,list=F,group = 50)
    train = analysisData_clean_nzv[split,]
    test = analysisData_clean_nzv[-split,]
    ```
## **Feature Selection**
Initially, I manually selected the variables I thought that were more relevant to Price regardless of the class of the variables, i.e. amenities. After data were transformed, I was left with 300+ variables. I then ran both the forward and backward selection process on the train dataset to select the most significant predictors. The predictors selected by the forward selection gave me a lower RMSE, so I decided to use the forward selection method.
    ```markdown
    start_mod = lm(price~1, data = train)
    empty_mod = lm(price~1, data = train)
    full_mod = lm(price~.,data = train)
    forwardStepwise = step(start_mod,
                       scope = list(upper=full_mod,lower=empty_mod),
                       direction = 'forward')

    summary(forwardStepwise)
    ```

## **Model Fitting**
I fitted different models, including Linear Model, Bagging, Random Forest, and Boosting. After I selected my predictors from forward selection, I decided to tune my model when I was fitting Random Forest and Boosting; however, my code never finished running after 8 hours or more. Eventually, I finally gave up and started tuning these models manually. As a result, boosting gave me the best model with the lowest RMSE.

During the fitting process, I found a few things that were quite interesting and I was not expected to see. After running the forward selection process, I had a set of predictors chosen by the process. Initially I decided to use the same set of predictors to fit all the models and see which model would give me the lowest RMSE. During the process, I noticed that removing outliers, imputing NAs, removing NAs, adding interactions significantly decreased the RMSE for linear regression but surprisingly, these actions increased the RMSE for both random forest and boosting. I then reverted most of the data cleansing I conducted and removed the interactions that I added for linear regression to fit the two models again. They indeed gave me lower RMSEs, a 60 for random forest and a 63 for boosting, than the lowest RMSE I got from linear regression, a 66. 

After reaching the lowest RMSE of my own record, I decided to tune the models manually since running CV was not an option for me due to slow performance. I tried to fit different numbers of trees for both random forest and boosting. I also tried fitting different interaction.depth, shrinkage, and cv.folds for boosting. In the end, boosting performed faster and returned a slightly lower RMSE, a 58, than what random forest returned, a 59.

  - Linear Regression
   ```markdown
    set.seed(617)
    model_LR_1 = lm(price ~.,train)
    pred1 = predict(model_LR_1, newdata = test)
    rmse1_test = sqrt(mean((pred1-test$price)^2)); rmse1_test
   ```
  - Regression Tree
   ```markdown
    set.seed(617)
    regTree1_complex = rpart(price~.,train,cp=0.001,method="anova")
    pred = predict(regTree1_complex,newdata=test)
    rmse_analysis = sqrt(mean((pred-test$price)^2)); rmse_analysis
   ```
   - Pruned Regression Tree
   ```markdown
    library(caret)
    trControl = trainControl(method="cv",number=10)
    tuneGrid = expand.grid(.cp=seq(0.001,0.1,0.001))
    set.seed(617)
    cvModel_1 = train(price~.,data = train,method="rpart",
              trControl = trControl,tuneGrid=tuneGrid)
    cvModel_1$bestTune

    treeCV_1 = rpart(price~.,data=train,
              control=rpart.control(cp=cvModel_1$bestTune))
    predTreeCV_1 = predict(treeCV_1,newdata=test)
    rmseCV_1 = sqrt(mean((predTreeCV_1-test$price)^2)); rmseCV_1
   ```
   - Bagging
   ```markdown
    library(randomForest)
    set.seed(617)
    bag = randomForest(price~.,data=train,mtry=ncol(train)-1,ntree=1000)
    predBag = predict(bag,newdata=test)
    rmseBag = sqrt(mean((predBag-test$price)^2)); rmseBag
   ```
   - Random Forest
   ```markdown
    library(randomForest)
    set.seed(617)
    model_RF_1 = randomForest(price ~.,data=train,ntree=1500)
    predForest = predict(model_RF_1,newdata = test)
    rmseForest = sqrt(mean((predForest-test$price)^2)); rmseForest
   ```
   - Random Forest with 10-fold Cross-validation
   ```markdown
    trControl = trainControl(method="cv",number=10)
    tuneGrid = expand.grid(mtry=1:42)
    set.seed(617)
    cvForest = train(price ~.,data=train,method="rf",ntree=10,trControl=trControl,tuneGrid=tuneGrid)
    predTreeCV_2 = predict(cvForest,newdata=test)
    rmseCV_2 = sqrt(mean((predTreeCV_2-test$price)^2)); rmseCV_2
   ```
   - Boosting
   ```markdown
    library(gbm)
    set.seed(617)
    boost = gbm(price ~.,data=train,distribution="gaussian",n.trees=5000,interaction.depth=3,shrinkage=0.01,cv.folds=5)
    pred_boost = predict(boost,newdata=test,n.trees=5000)
    rmse_boost = sqrt(mean((pred_boost-test$price)^2)); rmse_boost
   ```
   - Boosting with 10-fold Cross-validation
   ```markdown
    trControl = trainControl(method="cv",number=10)
    tuneGrid = expand.grid(n.trees=1000,interaction.depth=c(1,2),shrinkage = (1:100)*0.001,n.minobsinnode=5)

    garbage <- capture.output(cvBoost<-train(price ~.,data=train,method="gbm",trControl=trControl,tuneGrid=tuneGrid))
    boostCV = gbm(price ~.,data=train,distribution="gaussian",n.trees=cvBoost$bestTune$n.trees,
              interaction.depth=cvBoost$bestTune$interation.depth,
              shrinkage=cvBoost$bestTune$shrinkage,
              n.minobsinnode = cvBoost$bestTune$n.minobsinnode)
    predBoostCV = predict(boostCV,test,n.trees=1000)
    rmseBoostCV = sqrt(mean((predBoostCV-test$price)^2)); rmseBoostCV
   ```

## **Conclusions and Recommendations**


```markdown
Syntax highlighted code block

# Header 1
## Header 2
### Header 3

- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```

For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/verodhi/githubpagestest/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://help.github.com/categories/github-pages-basics/) or [contact support](https://github.com/contact) and we’ll help you sort it out.
