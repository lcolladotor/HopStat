So I am currently on a clinical trial and we have a very interesting export process.  Essentially, the data comes in an XML format, then is converted to data sets using SAS.  That's all fine and good, because SAS is good at converting things (I've heard).  The problem is, the people who started writing code to process the data and the majority of people maintaining the code use Stata.

Now that's a problem.  OK, well let's take these SAS data sets and convert them to Stata using Stat-Transfer.  Not totally reprehensible, it at least preserves some form versus a CSV to dta (Stata format) nightmare.  The only problem with this is that for some reason, the SAS parsing of the XML started to chew up a bit of memory, for about 140 data sets (it's a lot of forms).  Oh, by the way, a bit of memory was about *16 gigs* from a *100 meg* file. That's atrocious.  I don't care what it's doing but an 160 fold increase in just converting some XML and copying the datasets.  Not only that, it took *over 4 hours*. What the hell SAS?

Anyway, we just started a phase III trial collecting similar data from before. We're using the same database.  I decided to stop the insanity and convert the XML in `R`.  The data still needs to produce in Stata data sets, but at least I could likely control the memory consumption and the time limits.  At least I could throw it onto our computing cluster if things got out of control (I guess I could have done that with SAS, but I'm not doing that).  

Now, thank God that the `XML` package exists.  So pretty much just using some `xmlParse` and `xmlToDataFrame` commands, I had some `data.frame`s! Now, just make them into Stata data sets right? Not really.  

Let me first say that the `foreign` package is awesome.  You can read in pretty much any respectable statistical software dataset into `R` and do some exporting.  Also, the `SASxport` package allows you to export to the XPORT format using `write.xport`, which is a widely used (and [FDA-compliant](http://www.fda.gov/downloads/Drugs/DevelopmentApprovalProcess/FormsSubmissionRequirements/ElectronicSubmissions/UCM163561.pdf)) format.

Now what problems did I have with the `foreign` package?

1. I believe that Stata data sets can have length-32 variable names. 
      After some correspondence, the maintainers argue that Stata's
      documentation only support "up to 32" characters, which they
      interpret as only 31.  The 
      [documentation](http://www.stata.com/help.cgi?dta_113) states:

  > varlist contains the names of the Stata variables 1, ..., nvar, each up 
  > to 32 characters in length

  A week after my discussion, `foreign` had noted in their [ChangeLog](http://cran.r-project.org/web/packages/foreign/ChangeLog):
  >  man/{read,write}.dta: Freeze Stata support.

  Well, I guess I'll just change `write.dta` to do what I want.
    
  a. My solution: Copy `write.dta` and change the `31L` to `32L`.  Or moreover, I could have had the user pass a truncation length.  But     let's default some stuff.  The only concern is the command `do_writeStata` which is a hidden (non-exported) function from `foreign`.  So I just slapped a `foreign:::do_writeStata` on there, and away we go (not the best practice, but the only way I could think - `importFrom` did not work).  


2. Empty strings in `R` are represented as `""`.  According to the `foreign` package, Stata documentation states that empty strings is not supported, which is true:

  > Strings in Stata may be from 1 to 244 bytes long.
  
  and `""` has 0 bytes:
  
  ```r
  nchar("", "bytes")
  ```
  
  ```
  ## [1] 0
  ```


  I know from reading in Stata dta files, character variables can have data `""`, it's treated as missing.  See the Stata code below (I'm going to post about how to knit with Stata in followup)
  
  ```bash
  /Applications/Stata/Stata.app/Contents/MacOS/stata -b "test.do" &  
  cat test.log
  ```
  
  ```
  --------------------------------------------------------------------------------------------------------
        name:  <unnamed>
         log:  /Users/muschellij2/Dropbox/Public/WordPress_Hopstat/XML_to_Stata/test.log
    log type:  text
   opened on:  11 Jan 2014, 14:18:59
  
  . set obs 1
  obs was 0, now 1
  
  . gen x = ""
  (1 missing value generated)
  
  . count if mi(x)
      1
  
  . count if x == ""
      1
  
  . log close
        name:  <unnamed>
         log:  /Users/muschellij2/Dropbox/Public/WordPress_Hopstat/XML_to_Stata/test.log
    log type:  text
   closed on:  11 Jan 2014, 14:18:59
  --------------------------------------------------------------------------------------------------------
  ```

  This isn't a major problem, as long as you know about it.  
  
  a. My solution?  Make the `""` a `" "` (a space).  Is this optimal? No.  If there are true spaces in the data, then these are aliased.  But who really cares about them?  If you do, then change the code.  If not, then great, use my code.  If you have a large problem with that, and throw it in the comments and someone will probably read it.
  
Then, in Stata, you can define the function:

```r
*** recaststr- making the " "  to "" (to get around str0 cases)
capture program drop recaststr
program define recaststr
  foreach var of varlist * {
		local vtype : type `var'
		if ( index("`vtype'", "str") > 0) replace `var' = "" if `var' == " "
	}
end
```

and then run a `destring *, replace` command if you want.


OK, so if you ever want to use these functions, 


```r
require(devtools)
install_github("processVISION", "muschellij2")
library(processVISION)
```


should start you off: and the functions `write32.dta` and `create_stata_dta` should be what you're looking for.  I do some attempt at formatting the columns into numeric and dates in `create_stata_dta`.  If you don't want that, just use the argument `tryConvert=FALSE`.  Happy converting.

Followup Post: How to make a feeble attempt at knitting a Stata .do file and explaining some attempts/packages out there.
