Airbnb is a home-sharing internet platform that allows home-owners to market their properties online for short-term rentals. The renters, also called the hosts, are responsible for setting the prices on their own listings based on their intuition. Without a proper guidance on the pricing criterias, it could be very challenging for the hosts not to overprice their listings and at the same time selling short on their homes. Therefore, this project acts as the middle man by using several listing features to try fit a statistical model that predicts optimal prices of the listings for the hosts.


## Data Exploration

Two different sets of datasets were given, one was the analysis data and the other one was the scoring data, which was used for our submissions to Kaggle. The only difference between the two datasets were the number of observations and the additional Price variable in the analysis dataset. I started my data exploration process with the analysis dataset. This dataset contained 91 variables including Price, the outcome. It was challenging to manipulate the data without fully understood the variables, soI started eyeballing and exploring all the 90 variables one by one following the steps listed below:
 - Removed variables that were irrelevant to Price.   
 - Removed variables with high ratios of NAs.
 - Removed variables that contained only one value or one level.
 - Removed any variables that held redundant information.

## Data Transformation
After removing some variables through the manual process, I then conducted data transformation on some of the variables. 
1. Identified NAs and performed the following transformations on some variables.
   - Imputed NA values using the Caret package for security_deposit (35% were NAs) and cleaning_fee (16% were NAs). 
   - There were some 0s in Price, so I thought I would set those 0s to NAs and impute them with the Caret package in order to get better predictions.
   - Converted NAs in host_about to empty strings for future transformation.
   - I also removed 3 rows with number_of_reviews = 0 because there were few variables associated with reviews. With number_of_reviews being 0, all of the other variables were also NAs.
   - I then used rowSums to identify all the remaining NAs in the dataset and removed them. There were 15 of them left after performing the previous steps.
2. Converted characters to numeric values.
   - Some of the variables were factors with only two levels “t” or “f”, so I converted them into 1 for “t”s and 0 for “f”s and empty strings for the following variables:
      - Host_is_superhost
      - Host_has_profile_pic
      - Host_identity_verified
      - Require_guest_profile_picture
3. Converted class from integers to numeric.
   - Some of the variables were more suitable as numerics other than integers, so I converted the following variables:
      - Price
      - Extra_people
4. Transformed existing variables into new calculated variables.
   - Since the date fields were not meaningful in fitting models, I used them to calculate the length of hosting and length of listing in days and created new variables to store them.
   - The existing variable, host_local stored information of where the host was located. Since the hosts could be located in any location in the world, I chose to only look at the ones that were in New York, locally, and set them to 1, anywhere else 0. 
   - Prices could be affected for those hosts who provided some personal information, so I calculated the length of host_about and stored it in a new variable- host_about_len.
   - Amenities contained all the amenities the Airbnb provided into a string separated by “,”. I decided to clean the variable by removing all the spaces and separated all the amenities by “,”. After storing unique amenities into a list, I created variables for each of the amenities and assigned a 1 if that amenity was included in the unit and 0 if not.
   - Dummy variables- I didn’t want to include any categorical variables in my model, so I created dummy variables for all categorical variables and assigned them numeric values.
      - I created dummy variables for each level of the variables listed below and assigned a 1 if that level was presented in the row, if not, 0.
        -  Neighbourhood_group_cleansed
        -  Property_type
        -  Room_type
        -  Bed_type
        -  Cancellation_policy
        -  Neighbourhood_cleansed
   - After the above steps, I deleted the original variables from the dataset.
5. Reduced the gaps between values
   - The gaps between the prices of security_deposit were quite significant, so I decided to group the prices into a smaller scale of 0 to 10.
   
## Data Splitting
In order to predict the price of a listing, I used the process in the caret package to split data into train and test sets. Of which, train dataset consisted of 70% of the original data and 30% were split to test.

## Feature Selection
Initially, I manually selected the variables I thought that were more relevant to Price regardless of the class of the variables, i.e. amenities. After data were transformed, I was left with 300+ variables. I then ran both the forward and backward selection process on the train dataset to select the most significant predictors. The predictors selected by the forward selection gave me a lower RMSE, so I decided to use the forward selection method.

## Model Fitting
I fitted different models, including Linear Model, Bagging, Random Forest, and Boosting. After I selected my predictors from forward selection, I decided to tune my model when I was fitting Random Forest and Boosting; however, my code never finished running after 8 hours or more. Eventually, I finally gave up and started tuning these models manually. As a result, boosting gave me the best model with the lowest RMSE.

During the fitting process, I found a few things that were quite interesting and I was not expected to see. After running the forward selection process, I had a set of predictors chosen by the process. Initially I decided to use the same set of predictors to fit all the models and see which model would give me the lowest RMSE. During the process, I noticed that removing outliers, imputing NAs, removing NAs, adding interactions significantly decreased the RMSE for linear regression but surprisingly, these actions increased the RMSE for both random forest and boosting. I then reverted most of the data cleansing I conducted and removed the interactions that I added for linear regression to fit the two models again. They indeed gave me lower RMSEs, a 60 for random forest and a 63 for boosting, than the lowest RMSE I got from linear regression, a 66. 

After reaching the lowest RMSE of my own record, I decided to tune the models manually since running CV was not an option for me due to slow performance. I tried to fit different numbers of trees for both random forest and boosting. I also tried fitting different interaction.depth, shrinkage, and cv.folds for boosting. In the end, boosting performed faster and returned a slightly lower RMSE, a 58, than what random forest returned, a 59.

## Conclusions and Recommendations


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
