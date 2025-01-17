---
output: github_document
bibliography: paper/paper.bib
---

<!-- README.md is generated from README.Rmd. Please edit that file -->

# CTUcosting

Internal package for costings

```r
library(CTUCosting)
devtools::load_all()
```

## Run the app

```r
run_costing_app()
```



## Docker and app deployment

The app is deployed on CTUs shiny server. The shiny server requires creating a docker container with the app. 
The `docker/Dockerfile` file defines the code to create the docker container. 
This needs to be run on an ubuntu system (the server uses version 20.04, so that is a suitable version to use to build the container on. 
Set up a virtual machine with it). 
In order to make the container, clone this repository, copy the `Dockerfile` to the directory where you cloned this repository and 
create a `.Renviron` file with the REDCap token containing `CTUCosting_token = "xxxxxxxx"`, where `xxxxxxxx` is the REDCap token.
The directory should then contain (at least) 

    + CTUCosting
	|  + R
	|  |  + get_data.R
	|  |  + ...
	|  + inst
	|  + ...
	+ Dockerfile
	+ .Renviron

In the ubuntu VM, start a terminal from the above folder, and build the container via `sudo docker build -t ctucosting`. 
Test it via `sudo docker run --rm -p 3838:3838 ctucosting` and open the app in a guest OS browser by going to `localhost:3838`.

Create the docker image to be transferred to the shiny server via `sudo docker image save ctucosting | gzip > ctucosting-vx.y.z.tar.gz`.

Transfer the image to the shiny server via `scp ctucosting-vx.y.z.tar.gz username@shiny-02.ctu.unibe.ch:/home/shiny` 
(you need to have a log in to the server for this).



## Code example outside app

``` r
token <- Sys.getenv("CTUCosting_token")
record <- 4
costing <- 1

meta <- get_metadata(token = token)
d <- get_data(record = record, costing = costing, token = token)
info <- costing_info(d, meta$metadata)

wp <- get_workpackage_data(d, meta)
selected_workpackages <- summ_workpackages <- summarize_by_wp(wp)

selected_expenses <- expenses <- d$expenses %>% #names
  mutate(wp = sprintf("%05.1f", exp_pf)) %>%
  left_join(wp_codes(meta$metadata), by = c(wp = "val")) %>% #names
  left_join(redcaptools::singlechoice_opts(meta$metadata) %>% #names()
              filter(var == "exp_budget_pos") %>%
              select(val, lab) %>%
              mutate(val = as.numeric(val)),
            by = c(exp_budget_pos = "val")) %>%
  mutate(total_cost = exp_units * exp_cost) %>%
  relocate(Division = lab, Description = exp_desc, Amount = total_cost, wp_lab)

discount <- calc_discount(selected_workpackages,
              initcosting = info$initcosting,
              discount_db = info$discount_db)

overhead_tab <- overhead(selected_workpackages)

totals(workpackages = selected_workpackages,
       expenses = selected_expenses,
       discount = discount,
       overhead = overhead_tab,
       internal = info$internal,
       dlf = TRUE)

```
