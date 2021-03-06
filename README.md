# Introduction

**Designed and developed a movie recommender system based on Netflix user rating history.**

1.  Utilized data manipulation in Jupiter notebook to integrate and clean the raw data.

2.  Applied total 4 MapReduce jobs from preprocessing raw data to getting the final rating results.

3.  Implemented Item CF(collaborative filtering) to calculate and normalize co-occurrence matrix.

4.  Deployed MRUnit to test the MapReduce job logic and monitor the output.

*the data stored in repository here is sample data* 


## Data preprocessing
(Jupyter Notebook environment)

### visit the website

Crawled raw data from Netflix
Python using urllib
```python
from urllib import request
resp = request.urlopen('https://netflix.com/')
html_data = resp.read().decode('utf-8')
```
### extract from html
using beautiful soup ``` pip install BeautifulSoup```
to get the user_id, user_comments, movie_id, movie, user_rating
```python
BeautifulSoup(html,"html.parser")

from bs4 import BeautifulSoup as bs
soup = bs(html_data, 'html.parser')    
nowplaying_movie = soup.find_all('div', id='nowplaying')
nowplaying_movie_list = nowplaying_movie[0].find_all('li', class_='list-item') 

nowplaying_list = [] 
for item in nowplaying_movie_list:        
        nowplaying_dict = {}        
        nowplaying_dict['id'] = item['data-subject']       
        for tag_img_item in item.find_all('img'):            
            nowplaying_dict['name'] = tag_img_item['alt']            
            nowplaying_list.append(nowplaying_dict)
            
requrl = 'https://netflix.com/subject/' + nowplaying_list[0]['id'] + '/comments' +'?' +'start=0' + '&limit=20' 
resp = request.urlopen(requrl) 
html_data = resp.read().decode('utf-8') 
soup = bs(html_data, 'html.parser') 
comment_div_lits = soup.find_all('div', class_='comment')

eachCommentList = []; 
for item in comment_div_lits: 
        if item.find_all('p')[0].string is not None:     
            eachCommentList.append(item.find_all('p')[0].string)
```
### data cleaning
filter the useless rating history with spam comments, no comments, intended poor reviews, etc

filter the meaningless datas and only remain movie_id paired with movie, user_rating paired with user_id

Mainly using Pandas and regex(re library), numpy, etc

```python
comments = ''
for k in range(len(eachCommentList)):
    comments = comments + (str(eachCommentList[k])).strip()

import re

pattern = re.compile(r'[\u4e00-\u9fa5]+')
filterdata = re.findall(pattern, comments)
cleaned_comments = ''.join(filterdata)

import jieba    
import pandas as pd  

segment = jieba.lcut(cleaned_comments)
words_df=pd.DataFrame({'segment':segment})

stopwords=pd.read_csv("stopwords.txt",index_col=False,quoting=3,sep="\t",names=['stopword'], encoding='utf-8')#quoting=3全不引用
words_df=words_df[~words_df.segment.isin(stopwords.stopword)]
```


## MapReduce jobs

###  why choose Item CF
>User CF 
 a form of collaborative filtering based on this similarity between users calculated using people’s ratings of those items

>item CF
 a form of collaborative filtering based on this similarity between items calculated using people’s ratings of those items

1. the number of users weighs more than number of products
2. item will not change frequenctly
3. using user's historical data, more convicing


### Construct the mapreduce job queue and inject dependencies using maven.

Each job queue follows the walkthrough: 
1. import data from HDFS which stores data in block files
2. read and pair with the raw data through the mapper and the reducer to process the ra data
3. calculate co-occurrence matrix and normalize the co-occurrence matrix
4. matrix computation to get recommending result

## MRUnit test
MRUnit testing framework is based on JUnit and it can test Map Reduce programs written on Hadoop.

### sample test
655209;1;796764372490213;804422938115889;6
353415;0;356857119806206;287572231184798;4
835699;1;252280313968413;889717902341635;0

```java

import java.util.ArrayList;
import java.util.List;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mrunit.mapreduce.MapDriver;
import org.apache.hadoop.mrunit.mapreduce.MapReduceDriver;
import org.apache.hadoop.mrunit.mapreduce.ReduceDriver;
import org.junit.Before;
import org.junit.Test;
 
public class SMSCDRMapperReducerTest {
  MapDriver<LongWritable, Text, Text, IntWritable> mapDriver;
  ReduceDriver<Text, IntWritable, Text, IntWritable> reduceDriver;
  MapReduceDriver<LongWritable, Text, Text, IntWritable, Text, IntWritable> mapReduceDriver;
 
  @Before
  public void setUp() {
    SMSCDRMapper mapper = new SMSCDRMapper();
    SMSCDRReducer reducer = new SMSCDRReducer();
    mapDriver = MapDriver.newMapDriver(mapper);
    reduceDriver = ReduceDriver.newReduceDriver(reducer);
    mapReduceDriver = MapReduceDriver.newMapReduceDriver(mapper, reducer);
  }
 
  @Test
  public void testMapper() {
    mapDriver.withInput(new LongWritable(), new Text(
        "655209;1;796764372490213;804422938115889;6"));
    mapDriver.withOutput(new Text("6"), new IntWritable(1));
    mapDriver.runTest();
  }
 
  @Test
  public void testReducer() {
    List<IntWritable> values = new ArrayList<IntWritable>();
    values.add(new IntWritable(1));
    values.add(new IntWritable(1));
    reduceDriver.withInput(new Text("6"), values);
    reduceDriver.withOutput(new Text("6"), new IntWritable(2));
    reduceDriver.runTest();
  }
   
  @Test
  public void testMapReduce() {
    mapReduceDriver.withInput(new LongWritable(), new Text(
              "655209;1;796764372490213;804422938115889;6"));
    List<IntWritable> values = new ArrayList<IntWritable>();
    values.add(new IntWritable(1));
    values.add(new IntWritable(1));
    mapReduceDriver.withOutput(new Text("6"), new IntWritable(2));
    mapReduceDriver.runTest();
  }
}

```
Run the test class as JUnit class and it will pass or fail the test depending upon if the mapper is correctly written or not.

### counter test

One common use of self-created Counter is to track malformed records in the input.

```java
public class SMSCDRMapper extends Mapper<LongWritable, Text, Text, IntWritable> {
 
  private Text status = new Text();
  private final static IntWritable addOne = new IntWritable(1);
 
  static enum CDRCounter {
    NonSMSCDR;
  };
 
  /**
   * Returns the SMS status code and its count
   */
  protected void map(LongWritable key, Text value, Context context) throws java.io.IOException, InterruptedException {
 
    String[] line = value.toString().split(";");
    // If record is of SMS CDR
    if (Integer.parseInt(line[1]) == 1) {
      status.set(line[4]);
      context.write(status, addOne);
    } else {// CDR record is not of type SMS so increment the counter
      context.getCounter(CDRCounter.NonSMSCDR).increment(1);
    }
  }
  
public void testMapper() {
    mapDriver.withInput(new LongWritable(), new Text(
        "655209;0;796764372490213;804422938115889;6"));
    //mapDriver.withOutput(new Text("6"), new IntWritable(1));
    mapDriver.runTest();
      assertEquals("Expected 1 counter increment", 1, mapDriver.getCounters()
              .findCounter(CDRCounter.NonSMSCDR).getValue());
  }
}
```



