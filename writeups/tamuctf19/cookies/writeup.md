# Cookies

One of the web challenges really had me going crazy. When you 
log in you're presented with 2 web pages with a littany of images

![gigem.png](/blog/assets/cookie_images/gigem.png)

![cookies.png](/blog/assets/cookie_images/cookies.png)

Investigating the source will have you swimming in various permuted
strings trying to search for any semblence of a flag. 

The source revealed a clue in the Set-Cookie Header.
`Cookie: gigem_continue=cookies}; hax0r=flagflagflagflagflagflag; gigs=all_the_cookies; cookie=flagcookiegigemflagcookie`

However if you explored the image sources you'd be drowned in obfuscation. 

![image_sources.png](/blog/assets/cookie_images/image_sources.png)

I'll admit... this took way longer than it should have.

After some string parsing The gigem and cookies pages both had a single string revealing the following key:

Main page web source: `gigem{flag_in_`
Cookie page source: `gigem{continued == source_and_`
Cookie data: `gigem_continue=cookies}`
Flag: `gigem{flag_in_source_and_cookies}`

Aways remember: D.O.T.S

![note_to_self.png](/blog/assets/cookie_images/note_to_self.png)

Don't Over Think Shit
