---
title: 'RuCTFe 2012  Service gds writeup'
author: fish
layout: post
permalink: /ructfe-2012-service-gds-writeup/
categories:
  - CTF
  - writeup
tags:
  - RuCTFE
---
gds is a service coded in C++ and Qt. You may get the binary and source code from <http://ctftime.org/task/153/>.

The program locates at /home/gds/service and listens at TCP port 2012, and it receives some commands after you log in. Initially you may log in using booking/booking. Obviously we should patch it.

Keys are pushed into PNR database, which is located at /home/gds/pnrs.db. After some analysis of the binary under the help of IDA, we found a vulnerability that we could exploit. Simply type command \*RT \* after logging in, and you could retrive every single PNRs in database, including keys.

Now as we have the source code (I&#8217;m not on the mail list, so I found the code snippet right before I write this writeup, shiiiiiit), the vulnerability is more than obvious. The command *RT \* is first converted into a \*GDSCommand* instance, with no parameters (m_params is empty). Here is the definition of *GDSCommand*.

    class GDSCommand {
    public:
        GDSCommand(const QByteArray&);
        virtual ~GDSCommand() {};
        static QString help() { return "No help on subject"; }
        QString getParam( int i );
    protected:
        QStringList m_params;
    };
    

And let&#8217;s have a look at *GDSPnrDatabase::compareWithTemplate(const GDSPnr& f, const GDSPnr& templ)*:

    bool GDSPnrDatabase::compareWithTemplate( const GDSPnr& f, const GDSPnr& templ )
    {
        bool result = true;
        result &= ( templ.name.isEmpty() || f.name == templ.name );
        result &= ( templ.surname.isEmpty() || f.surname == templ.surname );
        result &= ( templ.passport.isEmpty() || f.passport == templ.passport );
        result &= ( templ.agent.isEmpty() || f.agent == templ.agent );
        result &= ( templ.uniqID.isEmpty() || f.uniqID == templ.uniqID );
        return result;
    }
    

The argument *templ* contains the argument in our command *RT <unique ID>*. The unique ID is retrived by *getParam(1);*, and *getParam()* is defined as follows:

    QString GDSCommand::getParam( int i )
    {
        return m_params.value(i);
    }
    

If *i* is out of bounds, then [a default-constructed value is returned][1]. Well, that&#8217;s why we could retrieve every PNRs by typing \*RT \*, where no parameters are provided.

BTW: We need a VPS in Korea or Russia for proxy next time.

 [1]: http://doc.qt.digia.com/qt/qlist.html#value