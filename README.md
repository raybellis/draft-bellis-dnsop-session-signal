# draft-bellis-dnsop-session-signal

If you want to have your system automatically rebuild the .xml and .txt
each time you save the file you will need nodejs and npm installed, and
then do:

    $ npm install

Edit `draft-bellis-dnsop-session-signal.md` using the hints found
[here](https://github.com/cabo/kramdown-rfc2629).  To view, run:

    $ grunt server

Otherwise, just use `kramdown-rfc2629` per above to convert the .md to
.xml, and then xml2rfc to convert that to .txt format.
