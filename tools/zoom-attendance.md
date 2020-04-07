---
layout: page
title: Merge Zoom Attendance Reports into Student Roster via Stata
---
This do-file takes attendance reports produced by Zoom and merges them into a student roster, recording for each date, for each student, the total number of minutes spent logged into Zoom, the number of log-ins, and whether the minutes exceeded a specified minimum for attendance.
The code assumes that the working directory has in it 1) A roster of students in Excel format (.xlsx) containing, along with any number of other columns, a column whose first-row cell has the words "Email Address"; 2) A set of one or more .csv files that come from Zoom, each a "report" from a class meeting. These .csv files should be named in some numerical or alphabetical sequence so that they will sort chronologically, e.g. "class01.csv" "class02.csv", etc. There should be no .csv files in the folder other than these report files. Note that this assumes students are logging in to Zoom using the email addresses in the roster; that is how records are matched. Stata will not modify the original roster file but will generate a new file (or over-write an existing file) containing the output, named "roster_with_attendance.xlsx". I wrote this for Stata 14.2.



## attend.do

```
* attend.do // Merge Zoom attendance reports into student roster

*** Constant

local minimum_minutes = 75 // student is considered to have attended if present on Zoom for this many minutes or more

*** Import master roster

set more off
clear

import excel using "roster.xlsx", firstrow case(lower)
ren emailaddress email
save "roster_with_attendance.dta", replace

*** Identify source files in current folder

global files: dir . files "*.csv" // Get a list of all .csv files in current folder
global files_sorted: list sort global(files) // Sort them
di in green `"$files_sorted"' // verify names of source files
local num_files : word count $files_sorted // Count source files

*** Iterate over source files

foreach f of global files_sorted {

* Import and basic cleaning of one source file

di _n "Processing: " as result "`f'"

clear

import delimited using "`f'", ///
  varnames(1) encoding("utf-8")

ren durationminutes minutes
label var minutes "Duration (minutes)"
ren alternativescore attention
label var attention "Attention score"

replace email = strlower(email) if email != strlower(email)

* Optionally, disregard instructors' email addresses
drop if email == "SOME_EMAIL_ADDRESS"
drop if email == "SOME_OTHER_EMAIL_ADDRESS"

* Processing of one source file

drop attention // Doesn't seem to vary
drop name // not needed

gen date = ustrleft(jointime,10)
label var date "Date"
drop jointime leavetime

local date_prefix = substr(date[1],1,2) + "_" + substr(date[1],4,2)
drop date

sort email
by email: egen mins_`date_prefix' = total(minutes)
label var mins_`date_prefix' "Total minutes attending Zoom"
by email: gen logins_`date_prefix' = _N
label var logins_`date_prefix' "Number of log-ins"
by email: keep if _n==1 // keeps one row for each email address
drop minutes // because have already captured total
gen byte attend_`date_prefix' = 0
replace attend_`date_prefix' = 1 if ///
  mins_`date_prefix' >= `minimum_minutes' & mins_`date_prefix' < .
label var attend_`date_prefix' "Attended for minimum # of minutes"

save temp_`date_prefix'.dta, replace

use roster_with_attendance.dta, clear
merge 1:1 email using temp_`date_prefix'.dta

di "Logins found for these email addresses that are not in the master roster:"
list email if _merge==2
drop if _merge==2
drop _merge

save roster_with_attendance.dta, replace
erase temp_`date_prefix'.dta

} // end of "foreach file" loop

recode mins_* logins_* attend_* (. = 0)

export excel using "roster_with_attendance.xlsx", replace

exit

*** Sample useful command
* list firstname lastname email if logins_04_02==0, clean noobs
```

Comments welcome. Posted April 4, 2020.<BR>
