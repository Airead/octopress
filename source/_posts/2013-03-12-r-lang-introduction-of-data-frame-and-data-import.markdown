---
layout: post
title: "R lang: introduction of data frame and data import"
date: 2013-03-12 19:17
comments: true
categories: R
---

In this post, we will introduct the operations of data frame, how to import data from external file. In addition there is a unusual example about importing data. Just for fun!

## Data Frames in R
Technically, a dataframe is R is a type of object. Less formally, a dataframe is a type of table where the typical use employs the rows as observations and the columns as variables. At first, We use R's data.frame command to create a dataframe and stored the result in the variable village.
<!--more-->
<code>
\> age = 18:29  
\> height = c(76.1,77,78.1,78.2,78.8,79.7,79.9,81.1,81.2,81.8,82.8,83.5)  
\> village = data.frame(age=age,height=height)  
</code>

Now we have data frame village, it looks like:  
<code>
\> village  
   age height  
1   18   76.1  
2   19   77.0  
3   20   78.1  
4   21   78.2  
5   22   78.8  
6   23   79.7  
7   24   79.9  
8   25   81.1  
9   26   81.2  
10  27   81.8  
11  28   82.8  
12  29   83.5  
</code>

The result is fairly clear. The contents of the vector age are stored in the first column of the dataframe under the heading "age" and the contents of the vector height are stored in the second column of the dataframe under the heading "height."

### Examining R's Workspace
Let's examine the contents of our workspace with R's ls() command (which stands for "list the contents of the workspace." The contents of your workspace might vary, depending on whether or not you have created other variables than those introduced in this activity.

<code>
\> ls()  
[1] "age"     "height"  "village"
</code>

Note that their are three objects in our workspace. We can examine the "class" of each object in our workspace, which tells us what kind of object is contained in each variable.

<code>
\> ls()  
\> class(age)  
[1] "integer"  
\> class(height)  
[1] "numeric"  
\> class(village)  
[1] "data.frame"  
</code>

Before we begin to explore dataframes in more depth, let's clean up our workspace a bit. Let's "remove" the age and height objects from our workspace.
`> remove(age,height)`

If you now try to print the contents of age and/or height, again you will see that they are gone.  
<code>
\> age  
Error: object "age" not found  
\> height  
Error: object "height" not found  
</code>  

Indeed, if you "list" the contents of your workspace, you will see that the age and height objects are gone. However, the dataframe village remains.

<code>
\> ls()  
 [1] "village"  
</code>

### Accessing the Variables in the Dataframe
We will now explain how to access the variables contained in our data frame. In this case, we know that the each column contains a variable.

<code>
\> village  
   age height  
1   18   76.1  
2   19   77.0  
3   20   78.1  
4   21   78.2  
5   22   78.8  
6   23   79.7  
7   24   79.9  
8   25   81.1  
9   26   81.2  
10  27   81.8  
11  28   82.8  
12  29   83.5  
</code>

The first column contains age and the second height. But we can quickly ascertain the names of the columns with R's names command. This would be useful if we were using someone else's dataframe and we were not sure of the column headings.

<code>
\> names(village)  
[1] "age"    "height"  
</code>

OK, as expected. But how do we access the data in each column? One way is to state the variable containing the dataframe, followed by a dollar sign, then the name of the column we wish to access. For example, if we wanted to access the data in the "age" column, we would do the following:

<code>
\> village$age  
 [1] 18 19 20 21 22 23 24 25 26 27 28 29  
</code>
 
In a similar fashion, we can access the values in the "height" column.

<code>
\> village$height  
 [1] 76.1 77.0 78.1 78.2 78.8 79.7 79.9 81.1 81.2 81.8 82.8  
[12] 83.5  
</code>

However, the additional typing required by the "dollar sign" notation can quickly become tiresome, so R provides the ability to "attach" the variables in the dataframe to our workspace.

`> attach(village)`  
Let's re-examine our workspace.

<code>
\> ls()  
 [1] "village"  
</code>

No evidence of the variables in the workspace. However, R has made copies of the variables in the columns of the dataframe, and most importantly, we can access them without the "dollar notation."

<code>
\> age  
 [1] 18 19 20 21 22 23 24 25 26 27 28 29  
\> height  
 [1] 76.1 77.0 78.1 78.2 78.8 79.7 79.9 81.1 81.2 81.8 82.8  
[12] 83.5  
</code>

It is important to understand that these are "copies" of the columns of the data frame. Suppose that we make a change to one of the entries of age.

<code>
\> age[1]=12  
\> age  
 [1] 12 19 20 21 22 23 24 25 26 27 28 29  
</code>
 
Note, however, that the data in the dataframe village remains unchanged.

<code>
\> village  
   age height  
1   18   76.1  
2   19   77.0  
3   20   78.1  
4   21   78.2  
5   22   78.8  
6   23   79.7  
7   24   79.9  
8   25   81.1  
9   26   81.2  
10  27   81.8  
11  28   82.8  
12  29   83.5  
</code>

## References
 - [Data Frames in R](http://msenux.redwoods.edu/math/R/dataframe.php)
