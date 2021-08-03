# The-new-R-pipe


The new pipe

The “pipe” is one of the most distinctive qualities of tidyverse/dplyr code. I’m sure you’ve used or seen something like this:

library(dplyr) 

mtcars %>%
   group_by(cyl) %>% 
   summarise(mpg = mean(mpg)) 
## # A tibble: 3 x 2
##     cyl   mpg
##    
## 1     4  26.7
## 2     6  19.7
## 3     8  15.1

That %>% is the operator that allows you to chain one function after another without the need to assign intermediate variables or deeply-nested parenthesis. Technically what it’s doing under the hood is evaluating the expression on the right-hand side fo the pipe (or, more usually, on the next line) using the expression on the left (or same line) as the first argument. The dplyr package depends on the magrittr package to do all that magic, and many other packages also import the magrittr pipe.

With version 4.1.0, it’s now possible to write

mtcars |>
   group_by(cyl) |>
   summarise(mpg = mean(mpg))
## # A tibble: 3 x 2
##     cyl   mpg
##    
## 1     4  26.7
## 2     6  19.7
## 3     8  15.1

What is the difference, other than one less character? (Not that the number of characters matters much if one uses the RStudio shortcut Ctrl + Shift + M. And with the new version of RStudio which is now in preview, one can choose which to use).

The main difference, for me, is that now you can use the pipe without relying on the magrittr package. Maybe this isn’t something you’ll lose sleep over, but as a rule of thumb it’s always desirable for your analysis to depend on as few different packages as possible. The more dependencies, the higher the probability of some update changing something important that destroys everything you built.

For those who use dplyr (or those maniacs that start their scripts with library(tidyverse)), |> and %>% are probably interchangeable. But there’s a whole multiverse outside the tidyverse. I, for example, prefer data.table to dplyr and my preferred syntax combines data.table with magrittr. With this change, I no longer need to start each script with library(magrittr) (although see the next section).

For those who have an (probably unhealthy?) obsession with speed and efficiency, |> appears to be faster than %>%. This is because magrittr does a lot of stuff behind the scenes, while the native pipe is just a syntax transformation. In other words,

x %>%
   mean()

is not literally equivalent to mean(x); there is a lot of processing behind those three characters. While

x |> 
   mean()

is interpreted by R exactly as mean(x). That is, there is zero overhead for using |>.

But the reality is that except in very special cases, the difference is negligible. In any worthwhile data analysis, the overhead of using magrittr is minuscule compared to the time it takes to do (and write!) the rest of the computations. My advice is not to get bogged down into chasing microsecond-level differences.

What you do need to pay attention to are the subtle (or not so subtle) differences between the two pipes. Perhaps the single most important difference is that the magrittr has a placeholder element for when one doesn’t want the left hand side result to go to the first argument of the right hand expression. The canonical example is the linear model:

mtcars %>% 
   lm(mpg ~ disp, data = .)
## 
## Call:
## lm(formula = mpg ~ disp, data = .)
## 
## Coefficients:
## (Intercept)         disp  
##    29.59985     -0.04122

Since the first argument of lm() is not the data, you have to use data = . to tell magrittr that mtcars does not have to be the first argument of lm(). The native pipe currently doesn’t have a placeholder. The way to fix that is to create an anonymous function:

mtcars |> 
   (function(x) lm(mpg ~ cyl, data = x))()
## 
## Call:
## lm(formula = mpg ~ cyl, data = x)
## 
## Coefficients:
## (Intercept)          cyl  
##      37.885       -2.876

This is quite ugly, so R’s answer is to use another trick from R 4.1.0: the new function-creation syntax. As of R 4.1.0 these two expressions are equivalent:

function(x) x + 1 
\(x) x + 1

The new “waving-person” syntax ( \(x)) essentially saves characters when creating functions. So by combining this with the native pipe, you can do

mtcars |> 
   (\(x) lm(mpg ~ disp, data = x))() 
## 
## Call:
## lm(formula = mpg ~ disp, data = x)
## 
## Coefficients:
## (Intercept)         disp  
##    29.59985     -0.04122

which is marginally more readable, though still quite ugly. The alternative syntax, which I think for now is a bit experimental, is this:

Sys.setenv(`_R_USE_PIPEBIND_` = TRUE) 

mtcars |> 
   . => lm(mpg ~ disp, data = .)
## 
## Call:
## lm(formula = mpg ~ disp, data = mtcars)
## 
## Coefficients:
## (Intercept)         disp  
##    29.59985     -0.04122

(Where the first line is indispensable and strongly signals that this code is not ready or prime time.)

So you first set the placeholder symbol (in this case .) and after the =>, you can write the same code that you would use in the magrittr pipe. In short, for cases where the . placeholder . is used, the replacement for %>% would be |> . =>. (Although, again, I understand that this syntax is neither definitive nor officially supported yet).
What about data.table?

Which brings me to my beloved data.table + magrittr syntax:

mtcars <- data.table::as.data.table(mtcars)
mtcars %>% 
   .[, .(mpg = mean(mpg)), by = cyl]
##    cyl      mpg
## 1:   6 19.74286
## 2:   4 26.66364
## 3:   8 15.10000

Given that the dot at the beginning of the first line is nothing less than magrittr’s placeholder, and that the new pipe has no placeholder, you might correctly guess that this syntax is not going to translate to the native pipe by simply changing %>% to |>. There are also some limitations, such as that the new pipe does not accept “special symbols” like [, + or * in the right expression.

Remembering the => syntax thiny, you would think that the proper translation would be to add |> . => but, alas, this is not the case:

mtcars |> 
   . => .[, .(mpg = mean(mpg)), by = cyl]
## Error: function '[' not supported in RHS call of a pipe

Ah, the limitation of those special symbols appears. Where do we go from here?

The trick is that R only looks at the function name, so all you have to do is rename the [ function (I love that everything in R is a function). For example, this code is perfectly cromulent:

.d <- `[` 

mtcars |> 
   .d(, .(mpg = mean(mpg)), by = cyl) 
##    cyl      mpg
## 1:   6 19.74286
## 2:   4 26.66364
## 3:   8 15.10000

Which is not so bad.
