# Prerequisites
- Probably have [hashcat](https://github.com/hashcat/hashcat) installed. I have it on some local directory on my Windows PC
- Figure out your hash type [NameThatHash](https://nth.skerritt.blog/). Usually for these its md5 (-m 500, is also default on hashcat but having it on makes it faster.)
# Loading passwords
Load the first two hashes (Easy) into a txt file like. Say the file is easy.txt, should just look like:
```
$1$abcd$cM9oWl3gP.t.Cz2dWWm7L1
$1$efgh$Fmv8YRcVm3phgfR0MwVYw0
$1$lmno$mDFxTnfk0Ad3di11sVbNj.
```
For taxonomoy:
```
$1$ -> Transaction ID ( i think )
$abcd$ -> Salt (prepended/appended to a password before feeding. Make it longer to password crack.
  Example: "password" is actually used for all 3 above, but salt makes them all different. <br>
  Thus if you wanted to crack "password", you'd have to try 3x as many combinatons.
$restofthestuff -> actual password hash
```
Command to generate: `openssl passwd -aixmd5 -salt abcd password`. Added `$1$` to front later.
I keep the real passwords out per NCL guidelines.
# Easy (passwords 1-2)
`hashcat.exe -m 500 easy.txt -a 0 path/to/rockyou.txt`  
-m 500 -> Mode for md5. Usually what they use for these problems, check NameThatHash.  
-a 0 -> attack mode 0, dictonary attack. This is default I think as well. Goes through wordlist(s).  
Format is generally "hashcat.exe [hashfile] [options] [dict1] [dict2] [dict...]  

If you read the problem, the password-maker is _really_ lazy and keeps the same pattern for all his passwords!(ex. password12 and 44password)  
They also use the _bare minimum_ number of characters required. Useful to limit your search.  
Now we have the keyword, we can start using more complex rules

# Medium Categories (Mask) (passwords 3-4)
Assuming hash file is in 'medium.txt'.   
This time they've updated their game and require 2 of:  
- Uppercase
- Lowercase
- Number
Luckily there two functions of hashcat that can help: mask attack and rules.
I would refer to official hashcat documentation on details but I'm here to just run you down to do the problem.
## Mask Attacks
With attack mode `-a 6` or `a -7`, you can try characters at end or beginning of a password.
| option | characters |
| --- | ---|
| ?d | 0123...9 |
| ?l | abcd...z |
| ?u | ABCD...Z |
| ?a | 01..9..AB..Z..ab..z..?!<>*$...~ |

So for example, say we have a file named `password.txt` that only contains:
```
password
PASSWORD
```
We can run `hashcat.exe -m 500 medium.txt -a 6 password.txt ?d?d` which will do `password11 -> password99` and `PASSWORD11 -> PASSWORD99`.  
We can reverse this with  `hashcat.exe -m 500 medium.txt -a 7 ?d?d password.txt`, which _prepends_ to beginning of word i.e `12password` and so on.  
If you'd like to combine categories, i.e. both digits and uppercase, we can assign custom codes with `-1 ?d?u` then instead we insert `password.txt ?1?1`.  
Everything behind `-1` gets added to a charcter list, so if you wanted only a certain set you could define it as `-1 ABCD1234` which will only try those 8 characters in any `?1` space.

## Rules
Can only be used with dictionary attacks (according to my machine, the wiki says something different). Does a lot of fancy things, like substitutions, duplications, case-toggles, etc, really over my head.   
Good ones to know:
[OneRuleToRuleThemAll](https://github.com/NotSoSecure/password_cracking_rules/blob/master/OneRuleToRuleThemAll.rule) and its [brother](https://github.com/stealthsploit/OneRuleToRuleThemStill)
Base hashcat rules: `rules/toggles1.rule` - `rules/toggles5.rule`,   `rules/leetspeak.rule`   

ORtRTA (?) takes long time but is mostly thourough, don't do unless you have lotta time.   
Toggle rules are easy and work really good for our next case (Medium+).   
Leetspeak will work with when NCL adds specifciation "Passwords must not contain any word from english dictionary".

## Solving the problem
I think I got it trying the ones below and masking sure `password.txt` had an uppercase KEYWORD and lowercase keyword:  
```
hashcat.exe -m 500 medium.txt -a 6 -1 ?d?u?l password.txt ?1?1  
hashcat.exe -m 500 medium.txt -a 7 -1 ?d?u?l ?1?1 password.txt
```  
Note_1: The amount of `?a` I add is depndent on minimum length and keyword length. They keyword itself was 6 characters so if the minimum was 10, I would add 4 `?d` to end like above.  
Note_2: Using custom `?d?u?l` since `?1` includes special characters which we won't use (problem states bare minimum safe to assume usually).
I was missing one from this category :(

### Addendum: Mask on front and back
Unforunately, you can't put both at front and end at same time, so we have to do it one at a time. hashcat allows for outputting your combinations to a file, so the sequence below would do it:
```
hashcat.exe  --stdout -a 6 password.txt ?a  > back.txt
hashcat.exe -m 500 medium.txt -a 7 ?a back.txt
```
This will try `1password1` all the way to `ZpasswordZ`. This could get the last one I'm missing. Also note for brevity I use `?a` here instead of the custom.
You can chain and combine different attack modes as well, like if I wanted to combine two wordlists `-a 1 first.name last.name > full.name` then do my append `-a 6 full.name ?d?d`. Not necessary here, but good to know.  

# Medium+ Categories+ (Rules) (passwords 5-6)
NCL now wants the passwords to have ALL of the categories, and a minimum length of 10.  
To get all the categories, I did something similar above and used hashcat rules to ALL capitalizations of my kEyWOrd:  
`hashcat.exe  --stdout -a 0 password.txt -r rules/toggles5.rule  > all.txt` (or type it by hand)  
Then I just ran what I basically did before but at a larger minimum:  
```
hashcat.exe -m 500 medium.txt -a 6 all.txt ?d?d?d?d  
hashcat.exe -m 500 medium.txt -a 7 ?d?d?d?d all.txt
```
I try digits first since the `all.txt` fulfills upper and lowercase already AND its faster to check. This alone got 2 passwords so safe bet. Otherwise you can use `-1 ?d?u?l` or whatever.

Got the one easy enough.
# Hard No English (password 7)
I just took everything from above and threw a rule on it since we can't have "english" but we can certainly have "3ng1!sh"
```
hashcat.exe --stdout -a 6 all.txt ?d?d?d?d > back.txt
hashcat.exe --stdout -a 7 ?d?d?d?d all.txt > front.txt
hashcat.exe -m 500 hard.txt -a 6 back.txt -r rules/leetspeak.rule
hashcat.exe -m 500 hard.txt -a 6 front.txt -r rules/leetspeak.rule
OR
hashcat.exe -m 500 hard.txt -a 6 front.txt back.txt -r rules/leetspeak.rule
```
I got my one and out. Happened to be in format `?d?d?d?dK3yW0rD`

# Hardest (passwords 7+)
I did not get to these but I think its combination of above plus:
- including special characters (leetspeek and ?a or ?s will w ork)
- higher minimum number (12)
I hear this is where it gets computationally prohibitive, so will take a while to try different orders and combinations.
