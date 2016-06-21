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

    run = (cfg) ->

      unless cfg.sender?
        debug 'Missing `sender`'
        return
      unless cfg.recipient?
        debug 'Missing `recipient`'
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

      socket.emit 'configure',
        support: true

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
      socket.on 'error', (o) ->
        debug 'connect', o.stack ? o.toString()
      socket.on 'disconnect', ->
        debug 'disconnect'

      socket.on 'report_dev', seem (event) ->
        subject = """
          #{event.error} in #{event.application} on #{event.host}
        """
        content = """

          Error `#{event.error}` was reported by #{event.application}
          on #{event.host} at #{event.stamp}.

          #{(JSON.stringify event, null, '  ').replace /^/gm, '    '}


          Thank you for your attention,
          #{pkg.name} (v#{pkg.version})

        """

        mail =
          from: cfg.sender
          to: cfg.recipient
          subject: subject
          markdown: content
        info = yield sendMail.call transporter, mail
        debug 'sendMail', info

    if require.main is module
      cfg = require process.env.CONFIG
      run cfg
