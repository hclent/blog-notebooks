
## A Year in (Code) Review

January is the time when people usually reflect on the past and create resolutions to improve themselves in the future. As such, I decided that this would also be a great time to **review some old code, identify weak spots, and build better coding habits for 2018**!

To start, I began by looking at some code pushed to [my github](https://github.com/hclent) in 2017. It didn't take long to pick up on one pervasive pattern in my old code: an unwarented affinity for `for` loops, especially nested ones! For example, if you look through my old code, you may find something like this:

```
interesting_data = defaultdict(lambda:0) #initalized a dict of some data I'm interested in counting

for item in some_list:
    for i in item:
        interesting_data[i] += 1 #add i to the dict as key, with count as value 
```

Note the nested `for` loops!! If you don't immediately see why this code is problematic, keep reading! In this post, I will explore **alternatives to `for` loops**, including **benchmarking**, and I will show you how reducing the amount of `for` loops makes code:

* More efficient
* More readable
* More functional (as in, functional programming)

This post will be **useful for intermediate programmers looking for ways to take their programming to the next level**. All of the code snippits will also be available as a Jupyter notebook [here]().


### Now let's get to some code

The code snippit I've chosen to review comes from my [Science Citation Knowledge Extractor]() tool. More specifically, it comes from a script called [journalvis.py](https://github.com/hclent/Science-Citation-Knowledge-Extractor/blob/master/flask/journalvis.py), which is used to generate a data visualization like this:

![journal vis](https://github.com/hclent/blog-notebooks/blob/master/images/journals.png)

This is an interactive data visualization that shows the distribution of articles published in academic journals by year. The top row shows the total number of articles published each year, and the subsequent rows show the total number of articles published in that particular journal each year when the user hovers over it. 

Here is the code being used to generate the top row:
```
for year in range(2000, 2018+1):
    yearDict[year]
    for pair in journal_year:
        if year == int(pair[1]):
            yearDict[year] +=1
```

For each year, the code iterates through all of the pairs to see if there are any places where the range year and the publication year match up. If so it adds it to our dictionary. There's gotta be a better way to do this!!

##  Its time to say goodbye to nested `for` loops!

First, let's import our dependencies. I will use `random`, `string`, and `numpy` to generate example data, `time` to do benchmarking, and `unittest` to do unit testing. `Plotly` will be used to graph our benchmarking results!


```python
import random, string, time, unittest
from collections import defaultdict
import numpy as np
import plotly
import plotly.plotly as py
import plotly.graph_objs as go
py.sign_in('#####', '######') #provide your username and API key to run Plotly in Jupyter!
```

Before we get started benchmarking the code containg nested `for` loop, we are going to need to generate some fake data that is similar to the data processed by `journalvis.py`. **Below are 3 functions to generate some random data**: one that generates fake journal names, another that generates random years, and a third function that combines each journal name with a random year.   


```python
def generateJournalNames(n):
    """
    Input: n (Int) number of fake journal names you want in the list.
    Output: a List containing n number of strings, 
    each string is 3 random capital letters in ABCD
    """
    list_of_journals = [(''.join(random.choice('ABCD') for _ in range(3))) for _ in range(n)]
    return list_of_journals
```


```python
def generateYears(n):
    """
    Input: n (Int) number of years (as Ints) that you want in the list. 
    (This will should match the the number of journals you generated).
    Output: a List containing n number of years (Ints) between 2000 - 2018.
    """
    years_list = list(np.random.choice(range(2000, 2018), n)) #np.random produces an array
    return years_list
```


```python
def generateData(n):
    """
    Input: n (Int) number of data points you want.
    Output: zip object containg (journal, date) pairs, representative of 
    the real world data processed by the original code
    """
    journals = generateJournalNames(n)
    years_list = generateYears(n)
    return zip(journals, years_list)
```

Now here is a function that will be used to benchmark runtimes using the inefficient, nested `for` loops:


```python
def generateDictInefficient(n):
    """
    Input: n (Int) number of journals & dates
    Output: Dict with year keys, and number values. 
    (E.g. {2000: 2, 2001: 1} means there were 2 articles published in journals in 2000,
    and 1 article published in a journal in 2001)
    ** This function is inefficient because of nested for loops! **
    """
    t0 = time.time()
    yearDict = defaultdict(lambda:0) #initialize empty dict

    #Generate random data
    journal_year = list(generateData(n)) #e.g. ('Scientific Reports', 2006)

    '''
    For each year from 2000 to 2018, look at each (Journal, Year) pair.
    If the year in the pair is the same as the year in range, +1 to the count in the dict.
    '''
    for year in range(2000, 2018+1):
        yearDict[year] #make sure each year is represented in the default dict
        for pair in journal_year:
            if year == int(pair[1]):
                yearDict[year] +=1
    
    #Keep track of time to see how inefficient this is
    t1 = time.time() - t0
    print("* finished in: " + str(t1))
    return yearDict
```

Let's try it out!


```python
yd0 = generateDictInefficient(10)
print("example year dict: ")
print(yd0)
generateDictInefficient(100)
generateDictInefficient(1000)
generateDictInefficient(10000)
generateDictInefficient(100000)
generateDictInefficient(1000000)
generateDictInefficient(10000000)
print("* done!")
```

    * finished in: 0.0003991127014160156
    example year dict: 
    defaultdict(<function generateDictInefficient.<locals>.<lambda> at 0x116879268>, {2016: 0, 2017: 0, 2018: 0, 2000: 0, 2001: 0, 2002: 0, 2003: 0, 2004: 0, 2005: 2, 2006: 0, 2007: 0, 2008: 1, 2009: 1, 2010: 2, 2011: 1, 2012: 1, 2013: 1, 2014: 1, 2015: 0})
    * finished in: 0.0017399787902832031
    * finished in: 0.014203071594238281
    * finished in: 0.09611797332763672
    * finished in: 0.9227039813995361
    * finished in: 9.228466987609863
    * finished in: 87.46316599845886
    * done!


For `n=10` (journal, year) pairs, its easy to count in the first example that all 10 examples were added into the dictionary, and thus the code is behaving as expected and has no bugs! But can I be so sure about the rest? Before we move on to explore alternatives to the nested `for` loops, I'm going to **add a test function to make sure there are no mistakes in the code that could be artificially inflating our benchmarking!**


```python
class TestCode0(unittest.TestCase):
    
    def test0(self):
        '''The resulting dictionary should have the same size as the int argument used 
        to create it'''
        result = generateDictInefficient(10000)
        result_size = sum(result.values())
        self.assertEqual(result_size, 10000)

```


```python
#configuration to execute tests in Jupyter notebook
if __name__ == '__main__':
    unittest.main(argv=['first-arg-is-ignored'], exit=False) 
```

    .

    * finished in: 0.09419417381286621


    
    ----------------------------------------------------------------------
    Ran 1 test in 0.095s
    
    OK


Great! Our code passed the test! Now let's see if we can produce the same results with some more efficient code. 

### First, I will try a **dictionary comprehension**:


```python
def generateDictEfficient1(n):
    """
    Input: n (Int) number of journals & dates
    Output: Dict with year keys, and number values. {2018: 1}
    """
    t0 = time.time()

    #Generate random data
    journal_year = generateData(n) #e.g. ('Scientific Reports', 2006)
    
    #pull out the years present in the data
    data_years = [pair[1] for pair in journal_year]
    
    #create a list of the years we are interested in counting
    all_years = [i for i in range(2000, 2019)]
    
    #dictionary comprehension
    yearDict = {y: data_years.count(y) for y in all_years}    
    
    #Keep track of time to see how inefficient this is
    t1 = time.time() - t0
    print("* finished in: " + str(t1))
    return yearDict
```


```python
yd1 = generateDictEfficient1(10)
print(yd1)
generateDictEfficient1(100)
generateDictEfficient1(1000)
generateDictEfficient1(10000)
generateDictEfficient1(100000)
generateDictEfficient1(1000000)
generateDictEfficient1(10000000)
print("* done!")
```

    * finished in: 0.00034999847412109375
    {2016: 0, 2017: 2, 2018: 0, 2000: 1, 2001: 0, 2002: 0, 2003: 2, 2004: 0, 2005: 2, 2006: 1, 2007: 0, 2008: 1, 2009: 0, 2010: 0, 2011: 0, 2012: 1, 2013: 0, 2014: 0, 2015: 0}
    * finished in: 0.0013248920440673828
    * finished in: 0.013056039810180664
    * finished in: 0.07970404624938965
    * finished in: 0.6227679252624512
    * finished in: 6.465986967086792
    * finished in: 62.540189027786255
    * done!


Before we compare `generateInefficient` to `generateDictEfficient1`, let's make sure that the latter passes a test. After all, it doesn't matter if `generateDictEfficient1` is faster if it is producing incorrect results! 


```python
class TestCode1(unittest.TestCase):
    
    def test1(self):
        '''The resulting dictionary should have the same size as the int argument used 
        to create it'''
        result = generateDictEfficient1(10000)
        result_size = sum(result.values())
        self.assertEqual(result_size, 10000)
```


```python
#configuration to execute tests in Jupyter notebook
if __name__ == '__main__':
    unittest.main(argv=['first-arg-is-ignored'], exit=False) 
```

    ..

    * finished in: 0.1074988842010498
    * finished in: 0.07501983642578125


    
    ----------------------------------------------------------------------
    Ran 2 tests in 0.184s
    
    OK


### Since `generateEfficient1` passes the test, now let's compare it to the nested `for` loops


```python
# Create traces
x_axis = [10, 100, 1000, 10000, 100000, 1000000, 10000000] 

inefficient = go.Scatter(
    x = x_axis,
    y = [0.00039, 0.00173, 0.014203, 0.096117, 0.9227, 9.22846, 87.4631],
    mode = 'lines+markers',
    name = 'inefficient'
)

efficient1 = go.Scatter(
    x = x_axis,
    y = [0.00034, 0.00132, 0.01305, 0.0797, 0.6227, 6.465, 62.5401], 
    mode = 'lines+markers',
    name = 'efficient1'
)

data=go.Data([inefficient, efficient1])

layout = dict(title = 'Benchmarking',
              xaxis = dict(title = 'N data points'),
              yaxis = dict(title = 'Time in seconds'),
              )
```


```python
fig0 = dict(data=data, layout=layout)
py.iplot(fig0)
```




<iframe id="igraph" scrolling="no" style="border:none;" seamless="seamless" src="https://plot.ly/~hclent/66.embed" height="525px" width="100%"></iframe>



As you can see, the nested `for` loops (blue) scale worse than the dictionary comprehension (orange). But in the case of our journal visualization, its unlikely that there will ever be a situation to visualize more than a few thousand journals. With `Plotly`, you can zoom in closer and see that there aren't any large gains for lower N values. I'm going to keep refining this code!

### Next, I will try creating a smaller, helper function to help make the code more efficient 

Similarly to the dictionary comprehension, it is preferable to only iterate through our pairs of `(Journal, Year)`s only once. Here, I will make a smaller function called `getUpdateValue`, which will allow me to figure out the current value for a giving year in the dictionary, add +1 for the year I'm looking at, then return size 1 dictionary, which I will use to `dict.update()` our yearDict in `generateDictEfficient2`. 


```python
def getUpdateValue(x, yearDict):
    """
    Input: x is a zip object (journal[String], year[Int]);
           yearDict is the most recent version of the dictionary. 
    Output: a dict containing the updated value for a particular year, e.g. {2000: 1}.
    This output dict will be used to UPDATE the yearDict of generateDictEfficient2().
    """
    year = x[1] #year is the second element of the zip obj
    value = yearDict[year] #retrieve the previous val of the yearDict for the year
    update_dict = {year: value+1} #add 1 to it
    return update_dict #return the current value for the year as a dict
```


```python
def generateDictEfficient2(n):
    """
    Input: n (Int) number of journals & dates
    Output: Dict with year keys, and number values. 
    """
    t0 = time.time()
    
    yearDict = {y: 0 for y in range(2000, 2018+1)} #dictionary comprehension to init dict
    
    #Generate random data
    journal_year = generateData(n) #e.g. ('Scientific Reports', 2006)
    
    '''
    First attempt to create our dict without a nested for loop.
    Although, we still have the for loop above to initliaze the dictionary...
    
    Here, I use a list comprehension that loops through each (journal, year) pair once. 
    
    This is better than the generateDictInefficient() method, which loops through
    all (journal, year) zip objects 18 times each! 
    
    For each zip object, the "getUpdateValue" function is called which:
    1) makes a copy of the current dictionary
    2) adds the next year count to an update_dict
    3) updates the values of yearDict with newly added values in update_dict
    '''
    update_dict = [yearDict.update( getUpdateValue(x, yearDict)  ) for x in journal_year]
    
    #Keep track of time to see how inefficient this is
    t1 = time.time() - t0
    print("* finished in: " + str(t1))
    return yearDict
            
```


```python
yd2 = generateDictEfficient1(10)
print(yd2)
generateDictEfficient2(100)
generateDictEfficient2(1000)
generateDictEfficient2(10000)
generateDictEfficient2(100000)
generateDictEfficient2(1000000)
generateDictEfficient2(10000000)
print("* done!")
```

    * finished in: 0.00020194053649902344
    {2016: 1, 2017: 2, 2018: 0, 2000: 0, 2001: 0, 2002: 0, 2003: 0, 2004: 0, 2005: 0, 2006: 1, 2007: 1, 2008: 0, 2009: 0, 2010: 1, 2011: 1, 2012: 1, 2013: 1, 2014: 0, 2015: 1}
    * finished in: 0.0006451606750488281
    * finished in: 0.005680084228515625
    * finished in: 0.06321096420288086
    * finished in: 0.5303189754486084
    * finished in: 5.539263010025024
    * finished in: 53.45106482505798
    * done!


### Before we compare `generateDictEfficient2` to the other methods, let's test it!


```python
class TestCode2(unittest.TestCase):
    
    def test2(self):
        '''The resulting dictionary should have the same size as the int argument used 
        to create it'''
        result = generateDictEfficient2(10000)
        result_size = sum(result.values())
        self.assertEqual(result_size, 10000)
```


```python
#configuration to execute tests in Jupyter notebook
if __name__ == '__main__':
    unittest.main(argv=['first-arg-is-ignored'], exit=False) 
```

    ...

    * finished in: 0.10398197174072266
    * finished in: 0.07726287841796875
    * finished in: 0.0549161434173584


    
    ----------------------------------------------------------------------
    Ran 3 tests in 0.239s
    
    OK


### It passes the test! So now let's compare:


```python
#Create a trace for generateEfficient2
efficient2 = go.Scatter(
    x = x_axis,
    y = [0.0002, 0.0006, 0.0056, 0.0632, 0.5303, 5.5392, 53.4510], 
    mode = 'lines+markers',
    name = 'efficient2'
)

data2 = go.Data([inefficient, efficient1, efficient2])

layout2 = dict(title = 'Benchmarking',
              xaxis = dict(title = 'N data points'),
              yaxis = dict(title = 'Time in seconds'),
              )
```


```python
fig1 = dict(data=data2, layout=layout2)
py.iplot(fig1)
```




<iframe id="igraph" scrolling="no" style="border:none;" seamless="seamless" src="https://plot.ly/~hclent/68.embed" height="525px" width="100%"></iframe>



By implementing the small helper function `getUpdateValue`, I managed to **reduce runtime by ~20 seconds** when `n=10million`! However between the dictionary comprehension (`efficient1`, orange) and the small helper function (`efficient2`, green), the later doesn't save us too much time. 

Now that we've seen dictionary comprehension and helper function solutions to this problem, I wanted to compare how much faster is a single `for` loop than the nested `for` loop? Are `efficient1` and `efficient2` much more scalable than just using a single `for` loop?


```python
def generateDictEfficientFor(n):
    """
    Input: n (Int) number of journals & dates
    Output: Dict with year keys, and number values. 
    """
    t0 = time.time()
    
    #Generate random data
    journal_year = generateData(n)
        
    yearDict = {y: 0 for y in range(2000, 2018+1)} #dictionary comprehension to init dict
       
    for jy in journal_year:
        yearDict[jy[1]] += 1 
    
    #Keep track of time to see how inefficient this is
    t1 = time.time() - t0
    print("* finished in: " + str(t1))
    return yearDict
```


```python
yd3 = generateDictEfficientFor(10)
print(yd3)
generateDictEfficientFor(100)
generateDictEfficientFor(1000)
generateDictEfficientFor(10000)
generateDictEfficientFor(100000)
generateDictEfficientFor(1000000)
generateDictEfficientFor(10000000)
print("* done!")
```

    * finished in: 0.00024008750915527344
    {2016: 1, 2017: 1, 2018: 0, 2000: 0, 2001: 1, 2002: 1, 2003: 0, 2004: 0, 2005: 0, 2006: 0, 2007: 2, 2008: 1, 2009: 1, 2010: 1, 2011: 0, 2012: 1, 2013: 0, 2014: 0, 2015: 0}
    * finished in: 0.0006899833679199219
    * finished in: 0.008743047714233398
    * finished in: 0.059123992919921875
    * finished in: 0.47548699378967285
    * finished in: 4.88400411605835
    * finished in: 48.08178186416626
    * done!


Let's test it...


```python
class TestCodeFor(unittest.TestCase):
    
    def test3(self):
        '''The resulting dictionary should have the same size as the int argument used to create it'''
        result = generateDictEfficientFor(100)
        result_size = sum(result.values())
        self.assertEqual(result_size, 100)
```


```python
#configuration to execute tests in Jupyter notebook
if __name__ == '__main__':
    unittest.main(argv=['first-arg-is-ignored'], exit=False) 
```

    ....

    * finished in: 0.09809994697570801
    * finished in: 0.07302403450012207
    * finished in: 0.05312514305114746
    * finished in: 0.0006167888641357422


    
    ----------------------------------------------------------------------
    Ran 4 tests in 0.228s
    
    OK


And compare it...


```python
#Create a trace for generateDictEfficientFor
oneForLoop = go.Scatter(
    x = x_axis,
    y = [0.0002, 0.0006, 0.0087, 0.0591, 0.47548, 4.8840, 48.0817],
    mode = 'lines+markers',
    name = 'generateDictEfficientFor'
)

data3 =go.Data([inefficient, efficient1, efficient2, oneForLoop])

layout3 = dict(title = 'Benchmarking',
              xaxis = dict(title = 'N data points'),
              yaxis = dict(title = 'Time in seconds'),
              )
```


```python
fig3 = dict(data=data3, layout=layout3)
py.iplot(fig3)
```




<iframe id="igraph" scrolling="no" style="border:none;" seamless="seamless" src="https://plot.ly/~hclent/70.embed" height="525px" width="100%"></iframe>



The single `for` loop wins here for speed. Although the dictionary comprehension and helper function only need to iterate through each data point once, the functions take longer because there is more pre-processing. For example, in `generateEfficient1` these pre-processing steps slow it down:

```
data_years = [pair[1] for pair in journal_year]
    
all_years = [i for i in range(2000, 2019)]
```
Here, the `for` loop has the benefit of being able to use the `+= 1` syntax, that is not available for dictionary comprehensions or list comprehensions. Having to only add to the values in a dictionary, rather than count items and then update a dictionary, is faster. 

But what about the readablility of the `for` loop? It still adds more lines to the code and therefore is less appealing. Again, since for such a data visualization as `journalvis.py`, there will probably not be a use case where 10million articles need to be accounted for. Up to 10k journals, the benchmarking time for `generateEfficient2` is very close to `generateEfficientFor`. 

## In conclusion 

Reviewing old code I wrote helped me to learn:

* To be more mindful of my code. Even if a piece of my code produces what I want, I should review that code again to see if I can make it more efficient and/or readable.
* Nesting `for` loops should be avoided and there are several other efficient, clean, and/or functional programming-based alternatives! 
    * Although its not the fastest, I thought `efficient1`'s use of the dictionary comprehension was the most readable.

Besides these main learning points, this code review also made me think about:

* Writing tests in Python. I frequently write tests in Scala, but have had little exposure to writing tests in Python. I would like to learn more about this!
* Adding docstrings to my code! Although I typically comment my code well, it is more likely that someone using my code will try to access the docstring for a method, than read the source code. 

Did you learn anything from this post? Do you have any ideas for alternative ways to approach this problem? Do I miss anything in this post? **If you would like to run any of this code, it is available on github [here]() as a Jupyter notebook.** 
