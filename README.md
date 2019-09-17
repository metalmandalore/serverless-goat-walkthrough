# serverless-goat-walkthrough
Verbose walkthrough


// color testing
colorize <- function(x, color) {
  if (knitr::is_latex_output()) {
    sprintf("\\textcolor{%s}{%s}", color, x)
  } else if (knitr::is_html_output())  {
    sprintf{"<font color='%s'>%s</font>", color, x)
  } else x
}
colorize("I wanna be RED", "red")
