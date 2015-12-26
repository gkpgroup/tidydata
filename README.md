# Explanation of the logic of run_analysis.R
Import all the libraries used by the R script 

      library(dplyr)
      library(stringr)
      library(reshape2)

## Reading reference data
This section of the code loads all the reference data files into memory

`Assumes: UCIHARDataset folder exist with all the text files as available from the download site.`

Here the `Activity Table` is read into data frame and proper column name is given

      act_tb <- read.fwf("UCIHARDataset/activity_labels.txt", widths = 21, sep = " ")
      colnames(act_tb) <- c("ActId", "ActName")

Next the `Features Table` is read and the second column is split to have relevant fields in separate column.

        features <- read.fwf("UCIHARDataset/features.txt", widths = 50, sep = " ")
        features<-cbind(features, read.table(text = as.character(features$V2), 
                                     sep = "-",fill = TRUE,strip.white = TRUE, 
                                     stringsAsFactors = FALSE))

Now that the columns are split, proper names are given to the columns

        colnames(features) <- c("FId", "ObsHeading", "Feature", "EstFunction","Axis")


Next few lines create a heading list for observations and formats it for ease of readability and use in queries.

        obsHeading <- as.character(factor(features$ObsHeading))
        obsHeading <- str_replace_all(obsHeading,"[()]","")
        obsHeading <- str_replace_all(obsHeading,"[-,]","_")

This placeholder string defines the observations that are needed for this problem. Having this defined helps in changing in needed in future and easy to use while filtering the data frame. Using this list let's subset the observations of interest and store it for later use.

        reqStr = c("mean()", "std()")
        features <- features[features$EstFunction %in% reqStr,]

Next few lines create a heading list for observations of interest and formats it for ease of readability and use in queries.

        interestedObsHeading <- as.character(factor(features$ObsHeading))
        interestedObsHeading <- str_replace_all(interestedObsHeading,"[()]","")
        interestedObsHeading <- str_replace_all(interestedObsHeading,"[-,]","_")

## Reading the train folder files
First read the list of subjects who participated in the survey and have the columns properly named.

        sub_train <- read.table("UCIHARDataset/train/subject_train.txt")
        colnames(sub_train) <- "Subject"

Next the Activity list is read and have the Activity description added to the data frame. Done a `Inner Join` to make sure all the records in the `y_train` table has to associated the Activity description.

        y_train <- read.table("UCIHARDataset/train/y_train.txt")
        colnames(y_train) <- "ActId"
        y_train <- left_join(y_train, act_tb)

Next is to read the Associated 561 observations for those activities and load them into a data frame. Have the data frame columns named properly using `obsHeading` list already prepared.

        x_train <- read.table("UCIHARDataset/train/X_train.txt")
        colnames(x_train) <- obsHeading

Now that all the `train` data sets have been read and formatted, they can be column binded together to get a consolidated data frame. 

        trainData <- cbind(sub_train,y_train,x_train)

It is good practice to release unwanted data frames to save memory

        rm(x_train, y_train, sub_train)

## Reading the test folder files
First read the list of subjects who participated in the survey and have the columns properly named.

        sub_test <- read.table("UCIHARDataset/test/subject_test.txt")
        colnames(sub_test) <- "Subject"

Next the Activity list is read and have the Activity description added to the data frame. Done a `Inner Join` to make sure all the records in the `y_test` table has to associated the Activity description.

        y_test <- read.table("UCIHARDataset/test/y_test.txt")
        colnames(y_test) <- "ActId"
        y_test <- left_join(y_test, act_tb)

Next is to read the Associated 561 observations for those activities and load them into a data frame. Have the data frame columns named properly using `obsHeading` list already prepared.

        x_test <- read.table("UCIHARDataset/test/X_test.txt")
        colnames(x_test) <- obsHeading

Now that all the `test` data sets have been read and formatted, they can be column binded together to get a consolidated data frame. 

        testData <- cbind(sub_test,y_test,x_test)

It is good practice to release unwanted data frames to save memory

        rm(x_test, y_test, sub_test)

##  Merging test and train data sets
Join the data from test and train folder (row bind) to have a single consolidated data frame.

        recordedData <- rbind(trainData, testData)

It is good practice to release unwanted data frames to save memory

        rm(testData, trainData)

## Generating the Tidy data set
Select only columns that are related to interested observations and order them by subhect and Activity Id (ActId)

        recordedData.select <- recordedData[,c("Subject", "ActId", 
                                      "ActName", interestedObsHeading)]
        recordedData.select <- arrange(recordedData.select, Subject, ActId)

To reshape the data, the consoliate data frome `recordedData.select` has to be melted. While applying this operation, first three columns will be set as ids.

        recordedData.molten <-  melt(recordedData.select,id=1:3)

Next get back the data frame to `wide form` using the `dcast` operation. During this operation also calculate the mean of the variables grouped by Subject and Act Id.

        tidy.data <- dcast(recordedData.molten,Subject + ActId + ActName ~ variable,
                            mean)

Prepare readable column names for the tidy data

        finalHeading <- c("Subject", "ActId", "ActName", 
                          paste("mean",interestedObsHeading,sep = "."))
        colnames(tidy.data) <- finalHeading

It is good practice to release unwanted data frames to save memory

        rm(recordedData, recordedData.molten, recordedData.select)

Last step is to save the tidy data set to a file in the working directory

        write.table(tidy.data,"tidydata.txt",row.names = FALSE)
