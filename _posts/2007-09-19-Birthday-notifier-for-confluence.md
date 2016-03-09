---
layout: post
title: "Birthday notifier for confluence"
date: 2007-09-19
---

To help me keep track of my friend's birthdays I've written a Confluence wiki page with their dates and a Python script that parses the page and sends an email.


The script looks for all confluence pages that have the label 'notifier', and parses a table with a format like this:


| Name      | Date       | user1@mydomain.com | user2@mydomain.com |
| Friend 1  | 1978-01-01 | x                  |                    |
| Friend 2  | 1975-01-01 |                    | x                  |


The table could contain many receiving email addresses, and a 'x' marks where to send the email. The Python script is installed in crontab and runs every night.


notifier.py


{% highlight python %}
#!/usr/bin/env python
# -*- coding: latin1 -*-

import MySQLdb
import datetime
import time
import smtplib
import quopri


class ParseError(Exception):
    pass

def parsetable(text):
    """Parses a text in confluence table format and returns a dictionary
    with header and content elements"""
    
    table_text = [line for line in text.split('\n') if line.startswith('|')]

    header = [item.strip('\ ') for item in table_text[0].split('||')[1:-1]]
    if len(header) < 3:
        raise ParseError, 'header'

    content = []
    for line in table_text[1:]:
        row = [item.strip('\ ') for item in line.split('|')[1:-1]]
        if len(row) != len(header):
            raise ParseError, 'content'
        if len(row[0]) > 0:
            content.append(row)

    return {'header': header, 'content': content}
        
def checknotify(table):
    """Iterates over a table (in the format that parsetable returns,
    and calls notify() if the date in the second row is near todays date"""
    
    today = datetime.date.today()
    for row in table['content']:
        subject = ''
        try:
            date=datetime.date(*time.strptime(row[1],"%Y-%m-%d")[:3])
        except ValueError:
            continue

        diff = datetime.date(today.year, date.month, date.day) - today
        name = quopri.encodestring(row[0])

        if diff.days == 5:
            subject = "%s soon %s years" % (name, today.year - date.year)
            text = "In five days %s will be %s =E5r.\n\nThe date was %s.\n" %(
                name, today.year - date.year, date)
        elif diff.days == 0:
            subject = "%s %s years today" % (name, today.year - date.year)
            text = "Today %s will be %s =E5r.\n\nThe date was %s.\n" %(
                name, today.year - date.year, date)
        if subject != '':
            dest = []
            for i in range(2, len(row)):
                if row[i] != '':
                    dest.append(table['header'][i])
            if len(dest) > 0:
                notify(subject, text, dest)

def notify(subject, text, destination):
    """Sends an email to destination"""
    
    sender = "noreply@localhost"
    message = "From: %s\r\n" % sender + \
              "To: %s\r\n" % ', '.join(destination) + \
              "Subject: Reminder: %s?=\r\n" % subject + \
              "Content-Transfer-Encoding: quoted-printable\r\n\r\n" + \
              text
    mailserver = smtplib.SMTP('localhost')
    mailserver.sendmail(sender, destination, message)
    mailserver.quit()

def main():
    """Connects to confluence and selects all pages that are marked with the
    label 'notifier', and sends email when a reminder is needed"""
    
    db = MySQLdb.connect(db="confluence",user="confluence",passwd="xxx")
    c = db.cursor()

    c.execute("select BODY from BODYCONTENT where CONTENTID
in (select CONTENTID from CONTENT_LABEL, LABEL where
CONTENT_LABEL.LABELID = LABEL.LABELID and LABEL.NAME =
'notifier')")

    for page in c.fetchall():
        try:
            table = parsetable(page[0])
            checknotify(table)
        except ParseError, detail:
            print "parse failed:", detail

    c.close()

if __name__ == "__main__":
    main()
{% endhighlight %}
