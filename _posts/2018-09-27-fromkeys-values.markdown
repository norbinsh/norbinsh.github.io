---
layout: post
title:  ".fromkeys() values"
date:   2018-09-27 22:37:20 +0300
categories: Python
---
Gist [here](https://gist.github.com/Norbinsh/734c0892934780b3e0c12c727c9b1e30).

So today I had to make sense out of a simple log file that contained transaction lines written by a logging package known
as [logzero](https://logzero.readthedocs.io/en/latest/).   

Snippet of said input file:
```
logzero_default - 2018-08-26 14:13:46,118 - INFO: user1@domain.com, 180, Version_1.zip
logzero_default - 2018-08-26 14:14:27,963 - INFO: user2@domain.com, 180, Version_1.zip
logzero_default - 2018-08-27 02:54:41,176 - INFO: user3@domain.com, 180, Version_1.zip
logzero_default - 2018-08-27 03:00:22,679 - INFO: user4@domain.com, 180, Version_2.zip
logzero_default - 2018-08-27 05:59:32,105 - INFO: user4@domain.com, 180, Version_2.zip
```

After reading the input file for all the rows, together with some regex, each line previously in the file is now 
stored as an object in a list ready to be accessed.

`
regex_pattern = r"^logzero_default - (\d{4}-\d\d-\d\d).*INFO: ([a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+), \d*. (.*).zip$"
`

```
def return_rows(input_file):
    with open(input_file, 'r') as inputf:
        return inputf.read().splitlines()

data_list = [re.split(regex_pattern, row) for row in return_rows(input_file)]
```

See the gist (link above) for the full code, it should be fairly straightforward - I am jumping straight to the interesting 
part:

Every time when I tried to iterate over the `data_list` and append data to it, I had seen strange results
(quantitative especially), and I could not figure at first what is going on - especially when I had thousands of them 
which made it harder to see.

`data_dict = dict.fromkeys(set(version[3] for version in data_list), dict.fromkeys(set(date[1] for date in data_list), []))`
  
For clarity sake, decided to work with a smaller data set, allowing me to actually see and understand what is going on.

I then realized, every change that I did in one of the lists, would show across all other keys in the dict:
First thing was to check the id() of the list, which is guaranteed to be unique and constant for each object - but you
gussed right - it wasn't unique. I got the same id() for any list I checked no matter under which key.

After some reading - it seems that the dict.fromkeys() method can't really "guess" whether the value you provide is 
meant to be mutable or not in order to make new copy of it every time, so the second value we pass into .fromkeys() is 
basically just a reference to the same object, which explains the above behaviour.

Changing to a nested dict comprehension seems to do the trick nicely! 

`data_dict = {ver: {date: list() for date in (set(date[1] for date in data_list))} for ver in versions_set}`

Now each list can be appended separately.




