---
title: "Europe is F***ing Up"
description: "New Cyber Resilience Act kills open source development in Europe"
date: "Jan 17 2024"
---

## The Background

The [Cyber Resilience Act] was passed by the EU recently, and it puts a slew of requirements and liabilities on developers.
In typical policy-maker fashion, the actual impact of the regulation will be the opposite of the intended effect. The goal from the EU regulators was to reduce cybersecurity vulnerabilites, 
but devs will just stop contributing to projects or leave Europe.

## Article 16

Straight from the law, *Article 16* deems an individual a manufacturer when they make significant changes to software, regardless of the license the software falls under.
What this means for open source developers: they are responsible for reporting, liability, and fines in the EU.

> A natural or legal person, other than the manufacturer, the importer or the distributor, that 
> carries out a substantial modification of the product with digital elements shall be considered 
> a manufacturer for the purposes of this Regulation.
> That person shall be subject to the obligations of the manufacturer set out in Articles 10 and 
> 11(1), (2), (4) and (7), for the part of the product that is affected by the substantial 
> modification or, if the substantial modification has an impact on the cybersecurity of the 
> product with digital elements as a whole, for the entire product.

## First Hand Scenario

I have recently made some pull requests that got merged into `Infisical SDK`, an open-source secret management software built in Rust, with language bindings to Python, Javascript, Swift, and more. 
I was interested in the project for the cryptography, but, if subject to this regulation, I wouldn't touch this project with a 10 foot pole. They deal with sensitive information that could leave them liable for damages in the millions of Euros if there was a serious vulnerability found, and they port to every language under the sun. I haven't made a dime from this project and probably never will. Yet, if they got fined, I could be on the hook. F*** that.

## Closing Thought

Europe is heading in the wrong direction. Less eyes will be on open source software in Europe, and more vulnerabilities will crop up as a result. They are reducing their talent pool and clearly don't understand the greatest strength of open projects: the right for anyone to freely contribute. 

[Cyber Resilience Act]: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=celex:52022PC0454