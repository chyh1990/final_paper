MAIN:=main
fast:
	xelatex $(MAIN)

all: 
	xelatex $(MAIN)
	bibtex $(MAIN)
	xelatex $(MAIN)
	xelatex $(MAIN)

open: fast
	open $(MAIN).pdf

distclean: clean
	rm -f main.pdf
clean:
	rm -f *.log *.thm *.toc *.out *.loe *.lof *.lot *.blg *.bbl
	find . -name '*.aux' -exec rm -f {} \;
