---
layout: page
title: Merge Zoom Attendance and Poll Reports into Student Roster, via Stata
---
(This was my first version. See [here for a new version](zoom-attendance-and-poll-quizzes.html) that scores correct answers to quizzes that the instructor gives via Zoom's poll feature, rather than merely using poll data to check on attendance.)

This do-file takes attendance and poll reports produced by Zoom and merges them into a student roster. For each Zoom lecture it determines, for each student, the total number of minutes spent logged into Zoom, the number of log-ins, and whether the student submitted at least one answer to Zoom polls. It then uses this information to calculate a simple 1-or-0 score for attendance that day. It also records attendance at weekly sections, which take place on various days. (It does not judge whether students' poll responses were "correct" or not, just whether any were submitted.)

It uses the student's email address to match Zoom reports with the roster. It will only work well if students are required to log into Zoom with the same email address that is in the roster file. To help check for situations where this is not the case, it generates a log file, "unknown_email.txt".

Stata will not modify the original roster file but will generate two new files (or over-write existing files) containing the output: "roster_with_attendance.dta" and "roster_with_attendance.xlsx", in Stata and Excel format.

The code assumes that zoomdata.do is in the working directory, which also has:

1. A folder named "rosters" which has in it "roster.xlsx". This is your master roster for the course, containing columns whose first-row cells have the words "Email Address" and "Section" respectively. This Excel file may also have any number of other columns, e.g. for students' names, etc.

2. A folder named "attend_lec", containing one or more .csv files that come from Zoom, each a "report" from a class meeting. These .csv files should be named in some numerical or alphabetical sequence so that they will sort chronologically, e.g. "class01.csv" "class02.csv", etc. There should be no .csv files in the folder other than these report files. Stata will extract the class date from the date-stamps in these files.

3. A folder named "poll_reports", containing one or more .csv files that come from Zoom, each a "report" from a class meeting. Zoom does not seem to reliably record dates in such files so you must embed the month and date in each filename as in the following example: poll04_07.csv

4. A folder named "attend_sec". The .csv files here are like those for lecture attendance above but each is for one section, rather than the whole class, and the sections take place on different days. They should be named "week01_A.csv", "week01_B.csv" "week02_A.csv", etc., for an arbitrary number of weeks (up to 99) and section names A, B, etc. Stata will compile these into a variable for number of minutes of section attended per week.

I wrote this for Stata 14.2. Comments welcome. (Updated 4/18/2020)


## zoomdata.do

```
* zoomdata.do // Merge data from Zoom attendance and poll reports into student roster

**************************************
***** Merge lecture attendance reports
**************************************

*** Constant

local min_minutes_lec = 75 // student is considered to have attended lecture if present on Zoom for this many minutes or more

*** Initiate log file for logins from unknown emails

file open uemail using unknown_email.txt, replace write text
file write uemail "Logins found for these email addresses that are not in the master roster:" _newline
file close uemail

*** Import master roster

set more off
clear

import excel using "rosters/roster.xlsx", firstrow case(lower)
ren emailaddress email

* The TA responsible for each student
gen ta = section
label var ta "Student's teaching assistant is"
order ta, after(section)
replace ta = "(TA name)" if ta == "01A"
replace ta = "(TA name)" if ta == "01B"
replace ta = "(TA name)" if ta == "01C"
replace ta = "(TA name)" if ta == "01D"

* Teaching assistants, who show up in Zoom data
gen byte is_ta = 0
label var is_ta "Is a teaching assistant"
order section id is_ta, first
local newobs = _N + 1
set obs `newobs'
replace firstname = "(TA name)" in `newobs'
replace lastname = "(TA name)" in `newobs'
replace email = "(TA email)" in `newobs'
replace is_ta = 1 in `newobs'

local newobs = _N + 1
set obs `newobs'
replace firstname = "(TA name)" in `newobs'
replace lastname = "(TA name)" in `newobs'
replace email = "(TA email)" in `newobs'
replace is_ta = 1 in `newobs'

save "rosters/roster_with_attendance.dta", replace

*** Identify source files in lecture attendance folder

cd attend_lec
global files: dir . files "*.csv" // Get a list of all .csv files in current folder
global files_sorted: list sort global(files) // Sort them

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
drop if email == "bread@ucsc.edu"

* Processing of one source file

drop attention // Doesn't seem to vary
drop name // not needed

gen date = ustrleft(jointime,10)
label var date "Date"
drop jointime leavetime

local date_prefix = substr(date[1],1,2) + "_" + substr(date[1],4,2)
drop date
local lecture_dates = "`lecture_dates'" + "`date_prefix'" + " "

sort email
by email: egen lec`date_prefix'_mins = total(minutes)
label var lec`date_prefix'_mins "Total minutes attending lecture"
by email: gen lec`date_prefix'_logins = _N
label var lec`date_prefix'_logins "Number of log-ins"
by email: keep if _n==1 // keeps one row for each email address
drop minutes // because have already captured total

save temp_`date_prefix'.dta, replace

use "../rosters/roster_with_attendance.dta", clear
merge 1:1 email using temp_`date_prefix'.dta

* Log emails that are not found in the master roster
count if _merge==2
local nonmatches = r(N)
if `nonmatches' > 0 {
  file open uemail using ../unknown_email.txt, write append text
  forvalues i = 1/`=_N' {
  	local flag = (_merge[`i']==2)
  	if `flag' == 1 {
      file write uemail "`f'  " (email[`i']) _newline
    }
  }
  file close uemail
}  // end if loop
drop if _merge==2
drop _merge

sort is_ta lastname firstname
save "../rosters/roster_with_attendance.dta", replace
erase temp_`date_prefix'.dta

} // end of "foreach file" loop

recode *_mins *_logins (. = 0)

cd .. // return to main folder
save "rosters/roster_with_attendance.dta", replace

*************************************************
***** Merge Zoom poll reports into student roster
*************************************************

*   Note: Zoom's poll output is inconsistent in whether it reports date
*     Need to put a date-stamp in filename of input files.

set more off

*** Identify source files in poll reports folder
cd poll_reports
global files: dir . files "*.csv" // Get a list of all .csv files in current folder
global files_sorted: list sort global(files) // Sort them

*** Iterate over source files

foreach f of global files_sorted {

* Import and basic cleaning of one source file

di _n "Processing: " as result "`f'"

clear

import delimited using "`f'", varnames(1) encoding("utf-8")
* import delimited using poll03.csv, varnames(1) encoding("utf-8")

ren useremail email

keep email // disregarding the content of the answers
replace email = strlower(email) if email != strlower(email)

* Processing of one source file

* Processing of date has to be different from attendance report because it's not consistently included
*   So, take from filename
local date_prefix = substr("`f'",5,5)

sort email
by email: gen lec`date_prefix'_answers = _N
label var lec`date_prefix'_answers "Number of poll answers"
by email: keep if _n==1 // keeps one row for each email address

save temp_`date_prefix'.dta, replace

use "../rosters/roster_with_attendance.dta", clear
merge 1:1 email using temp_`date_prefix'.dta

* Log emails that are not found in the master roster
count if _merge==2
local nonmatches = r(N)
if `nonmatches' > 0 {
  file open uemail using ../unknown_email.txt, write append text
  forvalues i = 1/`=_N' {
  	local flag = (_merge[`i']==2)
  	if `flag' == 1 {
      file write uemail "`f'  " (email[`i']) _newline
    }
  }
  file close uemail
}  // end if loop
drop if _merge==2
drop _merge

sort is_ta lastname firstname
save "../rosters/roster_with_attendance.dta", replace
erase temp_`date_prefix'.dta

} // end of "foreach file" loop

cd .. // return to main folder
recode *_answers (. = 0)
order *_mins *_logins *_answers, sequential
order section-email, first
save "rosters/roster_with_attendance.dta", replace

**********************************************************
***** Merge section attendance reports into student roster
**********************************************************

*** Identify source files in section attendance folder

cd attend_sec
global files: dir . files "*.csv" // Get a list of all .csv files in current folder
global files_sorted: list sort global(files) // Sort them

*** Iterate over source files

foreach f of global files_sorted {

* Import and basic cleaning of one source file

di _n "Processing: " as result "`f'"

clear

import delimited using "`f'", varnames(1) encoding("utf-8")

keep email durationminutes
ren durationminutes minutes
label var minutes "Duration (minutes)"
replace email = strlower(email) if email != strlower(email)

* Processing week # from filename
local week_prefix = substr("`f'",5,2)

sort email
by email: egen sec`week_prefix'_smins = total(minutes)
label var sec`week_prefix'_smins "Total minutes attending section"
by email: keep if _n==1 // keeps one row for each email address

drop minutes
save temp_`week_prefix'.dta, replace

use "../rosters/roster_with_attendance.dta", clear
merge 1:1 email using temp_`week_prefix'.dta, update

* Log emails that are not found in the master roster
count if _merge==2
local nonmatches = r(N)
if `nonmatches' > 0 {
  file open uemail using ../unknown_email.txt, write append text
  forvalues i = 1/`=_N' {
  	local flag = (_merge[`i']==2)
  	if `flag' == 1 {
      file write uemail "`f'  " (email[`i']) _newline
    }
  }
  file close uemail
}  // end if loop
drop if _merge==2
drop _merge

sort is_ta lastname firstname
save "../rosters/roster_with_attendance.dta", replace
erase temp_`week_prefix'.dta

} // end of "foreach file" loop

cd ..
recode *_smins (. = 0)
order *_mins *_logins *_answers *_smins, sequential
order section-email, first

*****************************************************
* Create variable marking attendance for each lecture
*   based on minutes online and poll participation
*****************************************************

foreach i in `lecture_dates' {
  gen byte lec`i'_attend = 0
  capture confirm variable lec`i'_answers    // check whether there was a poll that day
  if !_rc {                                  // if there WAS a poll that day
    replace lec`i'_attend = 1 if ///
      lec`i'_mins >= `min_minutes_lec' ///
      & lec`i'_mins < . ///
      & lec`i'_answers > 0 ///
      & lec`i'_answers < .
  }
  else {                                     // if there WASN'T a poll that day
    replace lec`i'_attend = 1 if ///
      lec`i'_mins >= `min_minutes_lec' ///
      & lec`i'_mins < .
  }
  label var lec`i'_attend "Attended for minimum # of minutes"
}

egen missed_classes = anycount(*_smins *_attend), values(0)
label var missed_classes "Total # of missed classes"

save "rosters/roster_with_attendance.dta", replace
export excel using "rosters/roster_with_attendance.xlsx", replace

*** Useful commands:
* browse ta first last *_smins *_attend if missed_classes > 4
* list ta first last if missed_classes > 4, clean noobs
```

