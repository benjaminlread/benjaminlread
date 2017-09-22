---
layout: page
title: Frequency Count of Zotero Tags via Stata
---
This do-file generates a list of tags (keywords) in a Zotero library. In the results, each observation contains a unique tag and a count of the number of times that tag appears in the Zotero library. Before running this do-file, you must first export your library from Zotero in .csv format (File --> Export library). Zotero contains two kinds of tags, "manual" (applied by the user) and "automatic" (retrieved from e.g. library databases). This code assumes you want to include both manual tags and automatic tags, but if you wish you can exclude one or the other type.

## tag_count.do

```
* tag_count.do
*   Creates dataset of tags with frequencies from exported Zotero database
*   Assumes Zotero library has been exported to "Library.csv" in current directory
*   Benjamin L. Read, http://benread.net

*** Import the library
import delimited using Library.csv, varnames(1) case(lower) clear
tostring manualtags, replace       // should already be strings but may not be e.g. if all null
tostring automatictags, replace
replace manualtags="" if manualtags=="."
replace automatictags="" if automatictags=="."

* One efficient approach would be to concatenate the "manualtags" and "automatictags" at the outset
*   Instead, we process the two types of tags separately in case that proves useful

*** Count each item's tags
* Manual tags
egen n_man_sc = noccur(manualtags), string(";") // Counts semicolon delimiters in tags string
gen int n_man_tags = n_man_sc + 1
replace n_man_tags = 0 if manualtags == ""
quietly summarize n_man_tags
local max_man_tags = r(max) // The maximum # of tags in any item
drop n_man_sc
* Automatic tags
egen n_auto_sc = noccur(automatictags), string(";") // Counts semicolon delimiters in tags string
gen int n_auto_tags = n_auto_sc + 1
replace n_auto_tags = 0 if automatictags == ""
quietly summarize n_auto_tags
local max_auto_tags = r(max) // The maximum # of tags in any item
drop n_auto_sc

*** Process manual tags
if `max_man_tags' > 0 {
  * Generate new string variables for the individual tags
  forvalues t = 1/`max_man_tags' {
    quietly gen str man_tag`t' = ""
  }
  * Loop through all observations and parse out the individual tags from the original string
  forvalues item = 1/`=_N' {              // Loop over all items
    local tokentarget = manualtags[`item']
    tokenize "`tokentarget'", parse(";") // double quotes needed around target, else commas produce error
    local ntokens = (n_man_tags[`item'] * 2) - 1 // delimiters are tokens too
    forvalues token = 1(2)`ntokens' {              // Loop over all tokens for this item
      local tag = (`token'+1)/2
      quietly replace man_tag`tag' = "``token''" in `item'  // Extract one tag
    }  // End token loop
  } // End item loop
  * Create new dataset in which each observation has one tag from one item
  preserve
  keep man_tag*
  rename man_tag* tag*
  gen newid=_n
  reshape long tag, i(newid) j(tagnum)
  drop if tag == ""
  save manual_tags, replace
  restore
}

*** Process automatic tags
if `max_auto_tags' > 0 {
  * Generate new string variables for the individual tags
  forvalues t = 1/`max_auto_tags' {
    quietly gen str auto_tag`t' = ""
  }
  * Loop through all observations and parse out the individual tags from the original string
  forvalues item = 1/`=_N' {              // Loop over all items
    local tokentarget = automatictags[`item']
    tokenize "`tokentarget'", parse(";") // double quotes needed around target, else commas produce error
    local ntokens = (n_auto_tags[`item'] * 2) - 1 // delimiters are tokens too
    forvalues token = 1(2)`ntokens' {              // Loop over all tokens for this item
      local tag = (`token'+1)/2
      quietly replace auto_tag`tag' = "``token''" in `item'  // Extract one tag
    }  // End token loop
  } // End item loop
  * Create new dataset in which each observation has one tag from one item
  keep auto_tag*
  rename auto_tag* tag*
  gen newid=_n
  reshape long tag, i(newid) j(tagnum)
  drop if tag == ""
  save automatic_tags, replace
}

*** Select manual tags, automatic tags, or both (as written, assumes both)
clear
cap use manual_tags             // assumes you want to include manual tags; comment this out otherwise
cap append using automatic_tags // assumes you want to include automatic tags; comment this out otherwise
cap erase manual_tags.dta       // clean up
cap erase automatic_tags.dta    // clean up

*** Collapse the dataset to create a per-tag count
gen freq=1
di as text _newline "Total number of tags: " as result _N
collapse (count) freq, by(tag) // creates new dataset; unit is a discrete tag
di as text _newline "Total number of unique tags: " as result _N
gsort - freq + tag // sorts in descending order of frequency. Comment this out to keep the tags in alphabetical order
di as text _newline "Top ten most frequent tags:"
list in 1/10 // type "list" without arguments for a full list, naturally
```

Comments welcome. Updated May 5, 2016.<BR>
