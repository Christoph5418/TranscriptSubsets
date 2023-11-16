# Subsetting Our Methods

I will explain the logic behind subsetting the number of exclusive transcripts for each method. I will also explain why I believe my problems are arising.


## General Logic

The general logic behind the code is pretty simple. I want to find the number of elements (namely transcripts) of every subset. I also want to make sure that each of these subsets have no overlap. 

This means, if our sets are A, B, and C, I want to determine exclusive subsets such that

$$A \cap AB = \emptyset \\ 
A \cap AC = \emptyset \\
A \cap ABC = \emptyset \\
AB \cap AC = \emptyset  \\
AB \cap ABC = \emptyset  \\
AC \cap ABC = \emptyset  \\
$$ 

The same logic should apply for subsets of B and C. 

If multiple sets have overlapping elements, the set with the most intersections should 'own' the element.

In essence, this would create the powerset. This means there would be  $2^n$ subsets.

### Pseudo Code

Given n sets, I first find the number of transcripts in each set.

I run gffcompare across all sets in round 1. I obtain every subset that comes from 2 of our original sets.
Ex. AB, AC, AD, AE, AF... BC, BD, BE, BF... CD, CE, CF...

**It does not matter what set I pick as the reference, because I am interested in matching, not novel.**

I run my matching transcripts script, so now we have the gtf of only the matching transcripts of the subsets listed above. 

**I must get rid of the novel transcripts (only keep matching), in case the next set I compare it to has matching novel transcripts.**
Ex. If I compare A and B, where A is reference. Then directly compare AB with C. AB.annotated will still contain all elements in B. C can still 'match' with class_codes that were NOT '='. This set would now contain elements not in A.

I run gffcompare with these sets with another set in round 2. Now I obtain every subset that comes from 3 of our original sets.
Ex. ABC, ABD, ABE, ABF...

I keep recursively doing this until I run out of sets that create unique subsets (I do not calculate BCA if ABC already exists for efficiency sake).

At each gffcompare, I grab the value of matching transcripts and store that value as the number of elements in the set.

I am left with every subset and the amount of elements in it, however, these subsets do not follow the exclusivity rule mentioned above.

The equations to find the exclusivity amounts is a simple formula to follow. This comes from subtracting elements that are in even more subsetted sets.

This would result in the amount of elements (transcripts) that are exclusive to each set.

## Subsetting at n = 3

I will walk you exactly what my subset3 script does.

**Input: 3 gtfs.**
**Output: Text file containing exclusive power set.**

**There are two methods I define:**

`count_transcripts() { awk '$3 == "transcript"' $1 | wc -l}`

Given a gtf, it returns the amount of transcripts in the file, by checking how many lines have 'transcript' in the third column.

`run_gffcompare() { gffcompare -o gffcomp_out_$1 -r $2 $3 awk '/Matching transcripts:/ {print $3}' gffcomp_out_$1.stats}`

Given an output name, gtf1, and gtf2, it runs gffcompare against the two gtfs and returns the amount of matching transcripts gffcompare finds.

**Now with those methods defined, I can explain the script.**

I firstly run count_transcripts with set A, B, and C. This gives me the amount of transcripts in each set.

I then call run_gffcompare to find the value AB, BC, and AC, alongside their annotated gtfs.

Now, I have the number of elements in A, B, C, AB, BC, and AC.

I filter AB to its matching transcripts, then call run_gffcompare with AB_matching and C, and store this value in ABC.

Now I have the value of A, B, C, AB, BC, AC, and ABC.

**From here I can find the 'exclusive sets.'**

To do this, I use these equations.

$$A_e = A - AB - AC + ABC \\
B_e = B - BC - AB + ABC \\
C_e = C - BC - AC + ABC \\
AB_e = AB - ABC \\
BC_e = BC - ABC \\
AC_e = AC - ABC\\
$$

And now I have the 'exclusive' sets.

### Explaining The Equations

<div align="center">
  <img src="https://www.conceptdraw.com/How-To-Guide/picture/3-circle-venn.png" alt="Venn Diagram" width="500" height="500">
</div>

Looking at this image of 3 sets, we can visualize these equations. 

ABC's value does not need to be changed, because there is no smaller set contained in it.

However AC, AB, and BC, have a smaller set in it. Thus, we must subtract ABC from each of these sets. This way we can get the exclusive set of AC, AB, and BC that does not include the elements in ABC.

We must do the same for A, B, and C. Looking at A, we would subtract AC, AB, and ABC. However inside AB and AC, there is ABC, so we must add 2 x ABC back in. This is where we get A = A - AB - AC + ABC.

## Subsetting at n = 4

There is where things get a little more tricky.

**Input: 4 gtfs.**
**Output: Text file containing exclusive power set.**

The approach is the exact same as subsetting with n = 3.

The script first finds the amount of transcripts in A, B, C, and D.

Then it finds AB, AC, AD, BC, BD, and CD.

Filters to the matching gtfs of these subsets.

Then finds ABC, ABD, ACD, and BCD.

Finds the matching of ABC.

Finds ABCD.

**From here, I can get the 'exclusive sets.'**

$$A_e = A - AB - AC - AD + ABC + ACD + ABD - ABCD \\
B_e = B - AB - BC - BD + ABC + ABD + BCD - ABCD\\
C_e = C - AC - BC - CD + ABC + ACD + BCD - ABCD \\
D_e = D - AD - BD - CD + ABD + ACD + BCD - ABCD \\
AB_e = AB - ABC - ABD + ABCD \\
BC_e = BC - ABC - BCD + ABCD \\
AC_e = AC - ABC - ACD + ABCD\\
AD_e = AB - ABD - ACD + ABCD \\
BD_e = BD - ABD - BCD + ABCD \\
CD_e = CD - ACD - BCD + ABCD \\
ABC_e = ABC - ABCD \\
ABD_e = ABD - ABCD \\
ACD_e = ACD - ABCD \\
BCD_e = BCD - ABCD \\
$$

### Explaining The Equations

<div align="center">
  <img src="https://www.mydraw.com/NIMG.axd?i=Templates/VennDiagram/Four-ellipseVennDiagram.png" alt="Venn Diagram" width="500" height="500">
</div>

I won't get into too much detail about the equations, however, they can be derived by looking at this venn diagram.

## My Problem

The problem I am getting occurs when I try to do subset with n = 4.

My output for some of the subsets are negative values - which is obviously wrong.

**I am fairly certain that the equations for exclusive sets is correct.**

This means the problem is either in my script or gffcompare.

After running the method and getting all of the values before finding the exclusive sets, I came across what might be causing the error.

According to the code, the set ABD had 12311 transcripts, while AB had just 12144. This obviously makes no sense.

After digging so more, I believe I came to what is causing the error.

I ran `gffcompare -r ~/bambu ~/bambu`, and checked the gffcmp.stats.

Query transcripts: 254642
Reference transcripts: 253094
Matching transcripts: 253094
Transcripts written into gffcmp.annotated: 254642.

I ran my matching filter on gffcmp.annotated, checked the amount of transcripts, and it said 254642.

This is most what is causing the error. For example, if AB = 253094 (since I grab matching transcripts form gffcmp.stats). Then I compare AB with D, if D has all that is in AB, then ABD would have 254642.

I believe that the error occurs in redundant transcripts in the gtf. When I ran bambu against itself, it said "1548 duplicate transcripts discarded". 254642 - 253094 = 1548.

And then when it compares the query to the reference, the 1548 extra transcripts in the query still get matched, because there is at least one copy of that transcript in the reference (the redundant got filtered out, but still one copy exists). 

This is what allows the filtered_matching annotated to have more transcripts than the amount that were claimed to be matching. So when I use annotated to compare with next set, it has more matching than I stored the value of.

**I believe to solve this problem I must first filter every set of transcripts to include no redundancy.**

Then in theory, the issue should be solved after this.

## Solution

I wrote a script called "remove_redundant.sh", which removes the redundant transcripts.

After running all of the gtfs with this, then running the subset method, all the numbers were positive.

In this example I got. I will compare them against before, when including the redundancy.


$$
Non \ redundant  \ on \ left\\
A_e = 80718: 82704 \\ 
B_e = 35150: 30\\
C_e = 30996: 30863 \\
D_e = 83868: 2728 \\
AB_e = 1071: -45 \\
BC_e = 65: 7 \\
AC_e = 537: 538\\
AD_e = 2798:  864\\
BD_e = 155180: 237745\\
CD_e = 317: 9 \\
ABC_e = 19: 42 \\
ABD_e = 3982: 12116\\
ACD_e = 69: 13\\
BCD_e = 342: 708 \\
ABCD_e = 120: 459 \\
$$

### Assessing Accuracy

To assess the accuracy I will explain the stats after checking redundancy:

*Ex: Amount with redundancy -  Amount after removing redundancy = Amount removed.x*
**Flair: 32639 - 32465 = 174 removed.**
**Stringtie: 96691 - 89313 = 7378 removed.**
**Human Sample: 251062 - 195929 = 55133 removed.**
**Bambu: 254642 - 246676 = 7966 removed.**

That is the amount before, after, and removed after finding redundant.

#### Looking at values of A

After removing redundant transcripts, the amount of total trancripts is 89313.

**Adding up all sets with A:** 80718 + 1071 + 537 + 2798 + 19 + 3982 + 69 + 120 = 89314 (one off).

Before removing redundant transcripts, the amount of total transcripts is 96691.

**Adding up all sets with A:** 82704 + -45 + 538 + 864 + 42 + 12116 + 13 + 459 = 96691 (same value).

This works for all sets, so this means that the equations are adding up correctly, meaning the equations are correct. Without removing redundancy, the values we get for each set does not make sense, since we get a negative value for AB.

This also suggests that the logic regarding comparing subsets across each other in the method is correct, otherwise these numbers wouldn't add up to the total transcripts.

Assuming that the redundant transcripts are removed correctly, I think the new results are correct-ish. I think the only limitation to how accurate it is depends on the accuracy of gffcompare.


