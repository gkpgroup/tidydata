## Explanation of the logic of run_analysis.R

Import all the libraries used by the R script 

<!-- -->
library(dplyr)
library(stringr)
library(reshape2)

## Reading reference data
This section of the code loads all the reference data files into memory 
Assumes: UCIHARDataset folder exist with all the text files as available from the download site.


# Activity Table - Reference 
<!-- -->
act_tb <- read.fwf("UCIHARDataset/activity_labels.txt", widths = 21, sep = " ")

# Give proper name to the columns
<!-- -->
colnames(act_tb) <- c("ActId", "ActName")

# Features Table - Reference
<!-- -->
features <- read.fwf("UCIHARDataset/features.txt", widths = 50, sep = " ")

# Split the data to have all the relevent fields in separate columns
<!-- -->
features<-cbind(features, read.table(text = as.character(features$V2), 
                                     sep = "-",fill = TRUE,strip.white = TRUE, 
                                     stringsAsFactors = FALSE))
# Give proper name to the columns
<!-- -->
colnames(features) <- c("FId", "ObsHeading", "Feature", "EstFunction","Axis")

# Create a heading list for observations
<!-- -->
obsHeading <- as.character(factor(features$ObsHeading))
obsHeading <- str_replace_all(obsHeading,"[()]","")
obsHeading <- str_replace_all(obsHeading,"[-,]","_")

# Data needed for this problem
<!-- -->
reqStr = c("mean()", "std()")

# Subset the features of interest
<!-- -->
features <- features[features$EstFunction %in% reqStr,]

# Create a heading list for observations of interest
<!-- -->
interestedObsHeading <- as.character(factor(features$ObsHeading))
interestedObsHeading <- str_replace_all(interestedObsHeading,"[()]","")
interestedObsHeading <- str_replace_all(interestedObsHeading,"[-,]","_")

## Reading the train folder files

# Subject detail
<!-- -->
sub_train <- read.table("UCIHARDataset/train/subject_train.txt")
colnames(sub_train) <- "Subject"

# Activity list
<!-- -->
y_train <- read.table("UCIHARDataset/train/y_train.txt")
colnames(y_train) <- "ActId"

# Have the Activity description added to the data frame
Done a `Inner Join` to make sure all the records in the `y_train` table has to associated the Activity description.
<!-- -->
y_train <- left_join(y_train, act_tb)

# Associated 561 observations for those activities
<!-- -->
x_train <- read.table("UCIHARDataset/train/X_train.txt")

# Get the observation heading updated for use later
<!-- -->
colnames(x_train) <- obsHeading

# Get all the training data together
<!-- -->
trainData <- cbind(sub_train,y_train,x_train)

# Release unwanted data tables to same memory
<!-- -->
rm(x_train, y_train, sub_train)

## Reading the test folder files

# Subject detail
<!-- -->
sub_test <- read.table("UCIHARDataset/test/subject_test.txt")
colnames(sub_test) <- "Subject"

# Activity list
<!-- -->
y_test <- read.table("UCIHARDataset/test/y_test.txt")
colnames(y_test) <- "ActId"

# Have the Activity description added to the data frame
Done a `Inner Join` to make sure all the records in the `y_test` table has the associated Activity description.
<!-- -->
y_test <- left_join(y_test, act_tb)

# Associated 561 observations
<!-- -->
x_test <- read.table("UCIHARDataset/test/X_test.txt")

# Get the observation heading updated for use later
<!-- -->
colnames(x_test) <- obsHeading

# Get all the training data together
<!-- -->
testData <- cbind(sub_test,y_test,x_test)

# Release unwanted data tables to same memory
rm(x_test, y_test, sub_test)

##  Merging test and train data sets
Join the data from test and train folder, order by Subject & activity id (ActId)
<!-- -->
recordedData <- rbind(trainData, testData)

# Release unwanted data tables to same memory
<!-- -->
rm(testData, trainData)

## Selection of interested observations
Select only columns that are related to interested observations
<!-- -->
recordedData.select <- recordedData[,c("Subject", "ActId", 
                                      "ActName", interestedObsHeading)]
recordedData.select <- arrange(recordedData.select, Subject, ActId)

## Now we are going to build the tidy data by melting and dcast

# Setting the ids as the first three columns
<!-- -->
recordedData.molten <-  melt(recordedData.select,id=1:3)

# Calculating the mean of variables grouped by Subject and Act Id
<!-- -->
tidy.data <- dcast(recordedData.molten,Subject + ActId + ActName ~ variable,mean)
finalHeading <- c("Subject", "ActId", "ActName", 
                  paste("mean",interestedObsHeading,sep = "."))
colnames(tidy.data) <- finalHeading

# Release memory of large data frames 
<!-- -->
rm(recordedData, recordedData.molten, recordedData.select)

## Write the data frame into a txt file
<!-- -->
write.table(tidy.data,"tidydata.txt",row.names = FALSE)
