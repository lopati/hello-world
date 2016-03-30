# -*- coding: utf-8 -*-
"""
Created on Sun Mar 27 20:36:53 2016

@author: kliu
"""
# https://github.com/mortada/fredapi (works for Tesla now!)
from fredapi import Fred

# https://research.stlouisfed.org/docs/api/api_key.html (free to use!)
fred = Fred(api_key='d94ebccb76f6251246594b7e1b5ee6cf')

# Matplotlib needed to display charts in console
import matplotlib.pyplot as plt

# Wrapper function to log 'users' in a text file
# For File I/O and Decorator requirement...
def loguser(func):
    def wrapper():
        x = func()
        fo = open('userlog.txt','a+') # More like a guest book?
        fo.writelines(x+';')
        fo.close()
    return wrapper

# Wrapper function to time how long it takes to search and draw charts
# From your decorator lecture; I just changed the formatting a little :P
def timeit(func):
    def wrapper(*arg):
        import time
        t = time.clock()
        res = func(*arg)
        print "\nThe time to run the function '{0}' was {1} seconds".format(
            func.func_name, time.clock()-t)
        return res
    return wrapper

@timeit
def search(searchID): # Search function utilizing fredapi fred.search() API
    try: # Returns only top 10 search results ranked by popularity
        x = fred.search(searchID, limit=10, order_by='popularity', 
                        sort_order='desc')
        print '\n{0}'.format(x['title']) # Displays just Series ID and Title
    except TypeError:
        print "\nNothing found. Please try again."

# Class to plot DataFrames returned by fredapi
class TimeSeries: # Class needs dataframe and title as arguments
    def __init__(self, dataframe, title):
        self.dataframe = dataframe
        self.title = title
 
    @timeit # The %chg. function needs user-supplied args for 'periods'
    def chgPlot(self,*args):
        x = self.dataframe.pct_change(*args)
        x.plot(kind='line',logy=False,secondary_y=False,legend=False,
               title=self.title)
    
    @timeit # For a log-scale y-axis
    def logPlot(self):
        self.dataframe.plot(kind='line',logy=True,legend=False,
                            title=self.title)
    
    @timeit # For just a plain old regular chart
    def rawPlot(self):
        self.dataframe.plot(kind='line',logy=False,legend=False,
                            title=self.title)

@loguser # Just a prompt to welcome the user (and log their activities ;)
def welcomeLogin():
    print '\nWelcome to a Front-end User Interface Test (FrUIT) \
    \nof the Federal Reserve Economic Database (FRED) API.'
    
    user = raw_input('Please enter your name to begin session: ')
    loopMenuItems() # Calls the main workhorse for the program
    
    print user
    return user # Enters into 'guestbook'

def loopMenuItems():
    loop = True # Sentinel value
    while loop:
        try: # Asks for user input  
            choice = int(raw_input("Please choose from the following menu items...\n[0] Quit FrUIT\t[1] Get series\t[2] Specify dates\n[3] Plot series\t[4] Plot %chg.\t[5] Export to CSV\n\nEnter choice: "))
        except ValueError:
            print "\nNot a valid entry. Please try again."
            continue
        
        if choice == 0: # Menu loop continues until Sentinel value exits
            print '\nGoodbye,'
            loop = False # Quit to welcomeLogin, which says bye to user and
                         #logs their stay through loguser decorator
        
        elif choice == 1: # Get series info if known or finds Series ID's if not
            parsedID = False # Sentinel value until valid 'series id' is parsed
            while not parsedID:
                ID = str(raw_input('Enter or look up Series ID: '))
                try: # Looks for series id and...
                    df = fred.get_series(ID) # If found, creates a DataFrame
                    info = fred.get_series_info(ID) # Also gets metadata
                    print '\n{0}'.format(info) # And prints on newline
                    parsedID = True # If all that happens, returns true and \
                                    #exits the inner loop
                except ValueError:
                    try: # If no series id found
                        search(ID) # Calls search function; hopefully finds \
                                   #what you're looking for...
                    except ValueError:
                        print "\nNot a valid entry. Please try again."
                        continue
                    continue
        
        elif choice == 2: # Gets specific dates if that's what you want
            parsedDate = False # Sentinel value until correct dates are parsed
            while not parsedDate:
                start = str(raw_input('Enter start date [yyyy-mm-dd]: '))  
                end = str(raw_input('  Enter end date [yyyy-mm-dd]: '))  
                try: # Checks dates
                    df = fred.get_series(ID,observation_start=start,
                                         observation_end=end)                       
                    parsedDate = True # Get out if true
                except ValueError:
                    print '\nThis is not a valid date range. Please try again.'
                    continue
                except UnboundLocalError:
                    print '\nPlease select a valid series before specifying dates.'                        
                    break
        
        elif choice == 3: # Time to start plotting!
            try: # Create instance of TimeSeries class to plot
                ts = TimeSeries(df,info['title']) # Pass it dataframe and title
            except NameError:
                print '\nPlease select a valid series before plotting.'
                continue
            log = raw_input('Log scale? (y/n): ').lower()        
            try: # Ask if you want a log scale or not; 'y' for yes...
                if log == 'y':
                    ts.logPlot()              
                    plt.show()      
                else: #everything else defaults to no.
                    ts.rawPlot()                  
                    plt.show() # Need this from Matplotlib to display in console
            except TypeError:
                print 'Sorry, no numeric data to plot.'
                continue
        
        elif choice == 4: # Option to plot % changes
            try: # Again, create instance of TimeSeries and pass df, title
                ts = TimeSeries(df,info['title'])
            except NameError:
                print '\nPlease select a valid series before plotting.'
                continue
            try: # Also pass additional 'periods' argument to calculate chg.
                periods = int(raw_input(
                'Select interval period to see %chg. over: '))       
                ts.chgPlot(periods)
                plt.show()
            except OverflowError:
                print '\nInterval invalid. Please try again.'
                continue
            except TypeError:
                print 'Sorry, no numeric data to plot.'
                continue
        
        elif choice == 5: # Find anything interesting? Export to CSV
            try:
                df.to_csv('fredAPI_output.csv') # Nice pandas' dataframe method
                print "\nTime series saved to 'fredAPI_output.csv'." 
            except UnboundLocalError:
                print '\nPlease select a valid series before exporting.'
                continue
            except IOError:
                print '\nPlease close file and try again.'
                continue
    
        else:
            print('\nPlease enter a value from 0 to 5.')

welcomeLogin() # Start program

def printUsageSummary(): # Some additional Python elements I can work in!
    import re
    userList = []
    cleanList = []
    
    fo = open('userlog.txt','r') # File I/O to access log data
    for line in fo.readlines():
        lineItem = line.strip().lower().split(';')
        userList.extend(lineItem)
    
    # Like Homework 3, but rewritten as list and dictionary comprehensions
    # note: regular expression used to clean up punctuation for weird logins
    cleanList = [re.sub(r'[^\w\s]','',x) for x in userList if x != '']
    uniqList = list(set(cleanList))    
    countDict = {i:cleanList.count(i) for i in uniqList}
    
    # Session summary info for user after they exit
    print '\np.s. Out of {0} session(s) and {1} user(s), you have used FRED FrUIT {2} time(s). That is all.'.format(
        len(cleanList), len(uniqList), countDict[cleanList[-1]])
    
    fo.close()

printUsageSummary() # That is all.
