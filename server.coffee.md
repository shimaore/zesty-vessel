Socket-to-Mail

    io = require 'socket.io-client'
    mailer = require 'nodemailer'
    direct = require 'nodemailer-direct-transport'
    smtp = require 'nodemailer-smtp-transport'
    {markdown} = require 'nodemailer-markdown'
    seem = require 'seem'
    Promise = require 'bluebird'
    request = (require 'superagent-as-promised') require 'superagent'

    pkg = require './package.json'
    @name = "#{pkg.name}:server"
    debug = (require 'debug') @name
    trace = (require 'debug') "#{@name}:trace"

    ignored_errors = [
      'invalid'
    ]

    run = (cfg) ->

Configuration:
- `sender` is the sending email address
- `dev_recipient` is the receiving email address(es) for `dev` messages
- `ops_recipient` is the receiving email address(es) for `ops` messages
- `csr_recipient` is the receiving email address(es) for `ops` messages
- `recipient` is the email address used if a more specific one is not available

      unless cfg.sender?
        debug 'Missing `sender`'
        return

      switch
        when cfg.mailer.SMTP?
          transport = smtp cfg.mailer.SMTP
        when cfg.mailer.DIRECT?
          transport = direct cfg.mailer.DIRECT
        else
          debug 'Missing `mailer`'
          return

      transporter = mailer.createTransport transport
      transporter.use 'compile', markdown
        gfm: true
        tables: true
      sendMail = Promise.promisify transporter.sendMail

      unless cfg.io?
        debug 'Missing `io`'
        return

      socket = io cfg.io

I had something more complex here that used the public API rather than the private one, but in our setup we don't allow machine authentication on the public API.

      socket.on 'connect_error', (o) ->
        debug 'connect_error', o.stack ? o.toString()
      socket.on 'connect_timeout', ->
        debug 'connect_timeout'
      socket.on 'reconnect', (n) ->
        debug 'reconnect', n
      socket.on 'reconnect_attempt', ->
        debug 'reconnect_attempt'
      socket.on 'reconnecting', (n) ->
        debug 'reconnecting', n
      socket.on 'reconnect_error', (o) ->
        debug 'reconnect_error', o.stack ? o.toString()
      socket.on 'reconnect_failed', ->
        debug 'reconnect_failed'

      socket.on 'connect', ->
        debug 'connect'
        socket.emit 'configure',
          support: true

      socket.on 'error', (o) ->
        debug 'connect', o.stack ? o.toString()
      socket.on 'disconnect', ->
        debug 'disconnect'

      subject_template = (event) ->
        switch event.error
          when 'missing-rule'
            """
              #{event.error} #{event.destination}
            """
          when 'not-registered'
            """
              #{event.error} #{event.destination}
            """
          when 'invalid'
            """
              #{event.error} #{event.ip}
            """
          else
            """
              #{event.error} in #{event.application} on #{event.host}
            """

      content_template = (event) ->
        """

          Error `#{event.error}` was reported by #{event.application}
          on #{event.host} at #{event.stamp}.

          #{(JSON.stringify event, null, '  ').replace /^/gm, '    '}


          Thank you for your attention,
          #{pkg.name} (v#{pkg.version})

        """

      log = (error) ->
        debug "error: #{error.stack ? error.toString()}"

      instantiate = (type) ->
        type_mattermost = "#{type}_mattermost"
        type_recipient = "#{type}_recipient"

        socket.on "report_#{type}", seem (event) ->
          return if event.error in ignored_errors

          channel = cfg[type_mattermost] ? cfg.mattermost
          if channel?
            yield request
              .post
              .send
                text: content
              .catch log

          recipient = cfg[type_recipient] ? cfg.recipient

          if recipient?
            subject = subject_template event
            content = content_template event

            mail =
              from: cfg.sender
              to: recipient
              subject: subject
              markdown: content
            info = yield sendMail
              .call transporter, mail
              .catch log
            trace 'sendMail', info

      for type in ['dev','ops','csr']
        instantiate type

    if require.main is module
      cfg = require process.env.CONFIG
      run cfg
