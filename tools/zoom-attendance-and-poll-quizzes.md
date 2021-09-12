---
layout: page
title: Process Zoom Output Files to Record Student Attendance and Quiz Performance, via Stata
---
In Spring 2021 I taught a class in which students attended lectures and sections via Zoom. In lectures, I also gave the students quizzes via Zoom's poll feature. Some of these were just-for-fun quizzes that merely piqued students’ interest or curiosity; students got credit for any answer (e.g., “Who came across as more convincing in the debate clips that we just watched?”). I call those "attention quizzes" because they reward students for paying attention. Other quizzes evaluated students’ grasp of the assigned reading (e.g., “Kharis Templeman argues that Taiwan’s indigenous people have not been as well represented as they could be and mainly blames ...”, a multiple choice question for which one of the several answer options is correct.)

This do-file takes attendance and poll reports produced by Zoom and merges them into a student roster. For each Zoom lecture it determines, for each student, the total number of minutes spent logged into Zoom. It then uses this information to calculate a simple 1-or-0 score for attendance that day. It also records attendance at weekly sections. It evaluates students' answers to the quiz questions and produces an overall score; it also generates a separate output file that can be uploaded to Canvas so students can see the number of quiz questions they got right, on a running basis. It also curves that score to integrate it into the students' grade. At the end it produces a file with an overall bonus/penalty for lecture attendance, and the curved quiz score. The end of the do-file I have appended several useful bits of code, for instance to generate a list of email addresses of those who have missed more than four lectures. (An earlier version of this do-file is [here](zoom-attendance.html)

It uses the student's email address to match Zoom reports with the roster. It will only work well if students are required to log into Zoom with the same email address that is in the roster file. To help check for situations where this is not the case, it generates a log file, "unknown_email.txt".

Stata will not modify the original roster file but will generate two new files (or over-write existing files) containing the output: "roster_with_zoom.dta" and "roster_with_zoom.xlsx", in Stata and Excel format respectively.

The code assumes that zoom.do is in the working directory, which also has:

1. A folder named "rosters" which has in it "roster.xlsx". This is your master roster for the course, containing columns whose first-row cells have the words "Email Address", "First Name" and "Last Name", in any order. This Excel file may also have any number of other columns.

2. A folder named "attend_lec", containing one or more .csv files that come from Zoom, each a "report" from a class meeting. These .csv files should be named in some numerical or alphabetical sequence so that they will sort chronologically, e.g. "lecture01.csv" "lecture02.csv", etc. There should be no .csv files in the folder other than these report files. Stata will extract the class date from the date-stamps in these files.

3. A folder named "attend_sec". The .csv files here are like those for lecture attendance above but each is for one section, rather than the whole class, and the sections take place on different days. They should be named "week01_A.csv", "week01_B.csv" "week02_A.csv", etc., for an arbitrary number of weeks (up to 99) and section names A, B, etc. Stata will compile these into a variable for number of minutes of section attended per week.

3. A folder named "poll_reports", containing one or more .csv files that come from Zoom, each a "report" from a class meeting, containing the answers that students gave to quiz questions delivered by poll. Zoom does not seem to reliably record dates in such files so you must embed the month and date in each filename as in the following example: 2021-03-30_lecture01.csv

4. A folder named "poll_questions", containing a file named "answer_key.csv". (I also put in this folder the .csv files containing the polling questions themselves for loading into Zoom, but this do-file does not use those.) The file answer_key.csv should start with the line "question,answer_correct,poll,anycorrect", and thereafter contain one line for each poll. For an "attention quiz" poll (for which any answer gets credit) the line looks like:

```
Can you speak or have you studied any of these languages?, ,2,1
```
Because any answer is correct, the second field is a single space, and "anycorrect" (the last field) is 1. The third field, "poll", is a number uniquely identifying this poll, in this case "2" for the 2nd poll of the term.

And for a "readings quiz" poll (for which there is one correct answer) the line looks like:

```
Who was Mao Zedong?,Communist Party leader and a founding father of the People's Republic of China,3,0
```
In such a question, the second field needs to exactly match the text of the correct answer option that you load into Zoom.

5. I also have a separate do-file containing all the excused absences I have granted for lectures. (Excusing a missed lecture does not also give students credit for any quizzes that day.) This do-file, "process_excused_absences.do", is made up of lines like "replace lec04_15_attend = 1 if firstname=="John" & lastname=="Smith". Zoom.do runs this do-file.

The line of that curves the quiz grades uses the function grade_curve from the "grade" package, which can be installed via: net install st0561.pkg.

I wrote this for Stata 14.2. Comments welcome. (Updated 9/11/2021)


## zoom.do

```
* zoom.do // Merge data from Zoom attendance and quiz poll reports into student roster

**************************************
***** Preliminaries
**************************************

*** Constants

local min_minutes_lec = 80 // student is considered to have attended lecture if present on Zoom for this many minutes or more
local min_minutes_sec = 45 // student is considered to have attended section if present on Zoom for this many minutes or more

*** Initiate log file for logins from unknown emails

file open uemail using unknown_email.txt, replace write text
file write uemail "Logins found for these email addresses that are not in the master roster:" _newline
file close uemail

*** Import master roster

set more off
clear

import excel using "rosters/roster.xlsx", sheet("Basics") firstrow case(lower)
keep id firstname lastname emailaddress
ren emailaddress email
label var firstname "First name"
label var lastname "Last name"
label var email "Email address"
save "rosters/roster_with_zoom.dta", replace

**************************************
***** Merge lecture attendance reports
**************************************

*** Identify source files in lecture attendance folder

cd attend_lec
global files: dir . files "*.csv" // Get a list of all .csv files in current folder
global files_sorted: list sort global(files) // Sort them

*** Iterate over source files

foreach f of global files_sorted {

  * Import and basic cleaning of one source file

  di _n "Processing attendance file: " as result "`f'"

  clear

  import delimited using "`f'", varnames(1) encoding("utf-8")

  ren durationminutes minutes
  label var minutes "Duration (minutes)"
  ren useremail email
  label var email "Email"
  ren nameoriginalname name_from_zoom
  label var name_from_zoom "Name from Zoom"
  drop guest

  replace email = strlower(email) if email != strlower(email)

  * Optionally, disregard instructors' email addresses
  drop if email == "[my email address here]"
  drop if email == "[TA's email address here]"

  * Processing of one source file

  gen date = ustrleft(jointime,10) // obtains date from first observation
  label var date "Date"
  drop jointime leavetime

  local date_prefix = substr(date[1],1,2) + "_" + substr(date[1],4,2)
  drop date
  local lecture_dates = "`lecture_dates'" + "`date_prefix'" + " "

  sort email
  by email: egen lec`date_prefix'_mins = total(minutes)
  label var lec`date_prefix'_mins "Total minutes attending lecture"
  by email: keep if _n==1 // keeps one row for each email address
  drop minutes // because have already captured total
  cap drop recordingconsent // as of late May 2021, reports contained this new variable

  save temp_`date_prefix'.dta, replace

  use "../rosters/roster_with_zoom.dta", clear
  merge 1:1 email using temp_`date_prefix'.dta

  * Log emails that are found in Zoom log but not found in the master roster
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

  sort lastname firstname
  save "../rosters/roster_with_zoom.dta", replace
  erase temp_`date_prefix'.dta

} // end of loop over each .csv of lecture attendance

drop name_from_zoom // it's a good idea to visually check that these match sometimes
recode *_mins (. = 0)
cd .. // return to main folder
save "rosters/roster_with_zoom.dta", replace

*** Create variable marking attendance for each lecture based on minutes online

foreach i in `lecture_dates' {
  gen byte lec`i'_attend = 0
  replace lec`i'_attend = 1 if lec`i'_mins >= `min_minutes_lec' & lec`i'_mins < .
  label var lec`i'_attend "Attended for minimum # of minutes"
}
order *_mins, last

*** Process excused absences
do attend_lec/process_excused_absences.do

egen missed_lec = anycount(*_attend), values(0)
label var missed_lec "Total # of missed lectures"
order missed_lec, after(email)

save "rosters/roster_with_zoom.dta", replace

**********************************************************
***** Merge section attendance reports into student roster
**********************************************************

*** Identify source files in section attendance folder

clear
cd attend_sec
global files: dir . files "*.csv" // Get a list of all .csv files in current folder
global files_sorted: list sort global(files) // Sort them

*** Iterate over source files

foreach f of global files_sorted {

  * Import and basic cleaning of one source file

  di _n "Processing: " as result "`f'"

  clear

  import delimited using "`f'", varnames(1) encoding("utf-8")

  cap rename useremail email // Zoom is inconsistent in label for this variable
  keep email durationminutes
  ren durationminutes minutes
  label var minutes "Duration (minutes)"
  replace email = strlower(email) if email != strlower(email)

  * Processing week # from filename, which looks like "Week04_A.csv"
  local week_prefix = substr("`f'",5,2)

  sort email
  by email: egen sec`week_prefix'_smins = total(minutes)
  label var sec`week_prefix'_smins "Total minutes attending section"
  by email: keep if _n==1 // keeps one row for each email address

  drop minutes
  save temp_`week_prefix'.dta, replace

  use "../rosters/roster_with_zoom.dta", clear
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

  save "../rosters/roster_with_zoom.dta", replace
  erase temp_`week_prefix'.dta

} // end of loop over each .csv of section attendance

sort lastname firstname
recode *_smins (. = 0)
order *_attend *_mins *_smins, sequential
order id firstname lastname email missed_lec, first
save "../rosters/roster_with_zoom.dta", replace
cd ..

* Create variable marking attendance for each section based on minutes online

foreach var of varlist *_smins {
  gen byte `var'_attend = 0
  replace `var'_attend = 1 if `var' >= `min_minutes_sec' & `var' < .
  label var `var'_attend "Attended for minimum # of minutes"
}

egen missed_sec = anycount(*_smins_attend), values(0)
label var missed_sec "Total # of missed sections"
order missed_sec, after(missed_lec)
drop *_smins_attend // comment this out if you wish to preserve the 0 or 1 attenance score for each section

save "rosters/roster_with_zoom.dta", replace

*************************************************
***** Merge Zoom poll reports into student roster
*************************************************

* Import answer key
* NOTE: Avoid commas and double quotation marks in questions and answers
* The macro pollkey will contain values of anycorrect, indictating whether a given poll
*   was what I call an "attention poll" (as long as a student gave an answer, any answer is correct)
*   or a "readings poll" (only one correct answer)
clear
cd poll_questions
import delimited using "answer_key.csv", varnames(1) encoding("utf-8")
gen str polltypestr = "_a" if anycorrect==1
replace polltypestr = "_r" if anycorrect==0
local n_polls = _N
local pollkey = ""
forvalues i = 1/`n_polls' {
  local temp = polltypestr[`i']
  local pollkey = "`pollkey'" + "`temp' "
}
save answer_key.dta, replace
clear
cd ..

*** Identify source files in poll reports folder
cd poll_reports
global files: dir . files "*.csv" // Get a list of all .csv files in current folder
global files_sorted: list sort global(files) // Sort them

*** Iterate over source files
cap erase poll_responses_cumulative.dta // just in case this file is there from previous runs
foreach f of global files_sorted {

  * Import and basic cleaning of one source file

  di _n "Processing poll report file: " as result "`f'"

  import delimited using "`f'", varnames(1) encoding("utf-8") clear

  ren useremail email
  keep email question answer
  replace email = strlower(email) if email != strlower(email)

  * Processing of one source file

  * Processing of date cannot be done the same was as in the attendance report
  *   because the date is not consistently included by Zoom
  *   So, take from filename
  local date_prefix = substr("`f'",5,5)

  merge m:1 question using ../poll_questions/answer_key.dta, ///
    keep(match) ///
    assert(match using) ///
    keepusing(answer_correct poll anycorrect)
  drop _merge

  * Either save this data as a new file, or append cumulative file to it
  capture confirm file poll_responses_cumulative.dta
    if _rc==0 { // if file exists, append current data to it
      append using poll_responses_cumulative.dta
      save poll_responses_cumulative.dta, replace
    }
    else { // if file does not already exist
      save poll_responses_cumulative.dta // saves current data to create file
    }

} // end of iteration through all poll report files

* Optionally, disregard instructors' email addresses in case they took the poll too
drop if email == "[TA email address here]"

* Score quizzes as correct or not and drop strings
gen byte score = answer == answer_correct
replace score = 1 if anycorrect==1 // if it's an attention poll with no wrong answers, score is 1
drop question answer answer_correct anycorrect

* Convert from long format (each obs is an email and one poll result)
*   to wide format (each obs is an email with many poll results)
sort email poll
reshape wide score, i(email) j(poll)
rename score* poll*

* rename score to reflect type of poll it was
forvalues i = 1/`n_polls' {
  local polltypestr `: word `i' of `pollkey''
  label var poll`i' "Score for poll # `i'"
  rename poll`i' poll`polltypestr'`i'
}

* Merge polls into main roster
save allpolls.dta, replace
use "../rosters/roster_with_zoom.dta", clear
merge 1:1 email using allpolls.dta, ///
  keep(match) ///
  assert(match using)
drop _merge
erase allpolls.dta

egen poll_a_total = rowtotal(poll_a*)
label var poll_a_total "Total for attention quizzes"
egen poll_r_total = rowtotal(poll_r*)
label var poll_r_total "Total for readings quizzes"
gen poll_total = poll_a_total + poll_r_total
label var poll_total "Total for all quizzes"
order poll_a_total poll_r_total poll_total, after(missed_sec)

cd .. // return to main folder
sort lastname firstname
save "rosters/roster_with_zoom.dta", replace

*************************************
*** Export quiz totals to file that can be uploaded to Canvas so students can see their running grade
*************************************

* Import and prepare Canvas Gradebook template for merge
clear
import delimited using "rosters/Politics 140d Spring 2021 - Canvas Gradebook Template.csv", varnames(1) encoding("utf-8")
ren id canvasid // id is 5-digit ID, specific to Canvas
ren sisuserid id // sisuserid is in fact UCSC ID. We will merge on this variable
save "rosters/canvas_template_temp", replace

* Merge
use "rosters/roster_with_zoom.dta", clear
merge 1:1 id using "rosters/canvas_template_temp"
drop if _merge==1 // student was in master not in canvas, presumably had dropped the class
drop _merge

* Keep relevant variables and rename them so they are right for Canvas
sort student
keep id student canvasid sisloginid section poll_r_total poll_a_total
ren id sisuserid // sisuserid is in fact UCSC ID
ren canvasid id // id is 5-digit ID, specific to Canvas
order student id sisuserid sisloginid section poll_a_total poll_r_total

* Output
*   Here we need specific items in the first line, containing spaces, so using "file write" instead of "export delimited", which
*   can only export variable names in the first line, not variable labels
*   Note the `"""' needed to write a double quote mark, i.e., a "
file open canvas_export using "rosters/grades-to-upload-to-canvas-poli-140d.csv", replace write text
file write canvas_export "Student,ID,SIS User ID,SIS Login ID,Section,Engagement Quiz Questions Correct,Reading Quiz Questions Correct" _n
forvalues i = 1/`=_N' {                                                  // Loop over all students
  local stuname
  file write canvas_export `"""' (student[`i']) `"""' "," (id[`i']) "," (sisuserid[`i']) "," ///
                           (sisloginid[`i']) "," (section[`i']) "," ///
                           (poll_a_total[`i']) "," (poll_r_total[`i']) _n
}     // End students loop
file close canvas_export

*************************************
*** Final adjustments; export to Excel spreadsheet and leave full dataset in memory
*************************************

clear
use "rosters/roster_with_zoom.dta"

* For overall quarter grade calculation: Curved poll grade, and lecture attendance factor
egen poll_curve = grade_curve(poll_total), mean(80) max(100) // uses "grade" package, net install st0561.pkg
label var poll_curve "Quiz grades, curved (0-100)"
order poll_curve, after(poll_total)
* histogram poll_curve, width(1) freq xlabel(60(10)100)
gen byte missed_lec_adj = -1 * (missed_lec - 2) // last number is the number of freebies students get (unpenalized missed classes)
label var missed_lec_adj "Adjusted missed lectures" // positive for bonus, negative for penalty
gen byte lec_bonus = 0
label var lec_bonus "Bonus/penalty for lecture attendance"
order missed_lec_adj lec_bonus, after(missed_lec)
replace lec_bonus = missed_lec_adj * 0.5 if missed_lec_adj > 0 // positive bonus
replace lec_bonus = missed_lec_adj * 2 if missed_lec_adj < 0 // negative bonus, i.e., penalty

* Export four variables for overall quarter grade calculation in Google Docs
export delimited id lastname lec_bonus poll_curve using "rosters/grades-for-gdoc-poli-140d.tsv", delimiter(tab) replace

* Export to Excel
export excel using "rosters/roster_with_zoom.xlsx", replace

exit

*** Some useful commands:

* Inspect data of those who have missed a lot of lectures
browse firstname lastname *_smins *_attend if missed_lec > 4

* Output a list of those who have missed a lot of lectures
list first last if missed_classes > 4, clean noobs

* Output the email addresses of those who have missed a lot of lectures
list email if missed_classes > 4, clean noobs

* Show the mean scores on all reading quizzes
su poll_r*

* Visualize distribution of quiz grades, and curve them. See also above.
histogram poll_r_total, discrete frequency width(1)
egen curve = grade_curve(poll_r_total), mean(83) max(100) // uses "grade" package, net install st0561.pkg
histogram curve, width(1) freq xlabel(60(10)100)

* List which classes a student missed
gen long obsn = _n 
su obsn if last == "Name_here", meanonly 
local target_obsn = `r(min)' // target_obsn now is the number of the observation for the student in question
foreach var of varlist *attend {
  if `var'==0 in `target_obsn' {
    di "`var'"
  }
}
drop obsn

* Create and graph a new dataset showing attendance totals per lecture
collapse (sum) *attend
rename *_attend *
xpose, clear varname
ren _varname date
ren v1 attendance
gen byte lecture = _n
twoway line enroll acadyear, yscale(range(0 45)) ylabel(0(5)45, grid) ytitle("Number of students attending", margin(r=2)) xtitle("Lecture number", margin(t=2)) title("Attendance in Politics 140d lectures", margin(b=2))
```

