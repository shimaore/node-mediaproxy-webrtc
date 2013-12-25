    module.exports = (config) ->

Obviously `getUserMedia()` is useless, since we will have multiple users (and therefor user streams).
So we provide a dummy implementation for `MediaStream` and `getUserMedia()`.

See http://www.w3.org/TR/mediacapture-streams/#idl-def-MediaStream

      class MediaStream
        constructor: ->
          @id = new uuid();

          # @onended = null;
          # @onaddtrack = null;
          # @onremovetrack = null;

        stop: ->
          # Do nothing, who would we talk to?

      #  MediaStream.prototype.getAudioTracks
      #  MediaStream.prototype.getVideoTracks
      #  etc.: not implemented
      #  Doesn't look like we're using any other attribute.

For `getUserMedia` see http://www.w3.org/TR/mediacapture-streams/

      getUserMedia = (options,success,failure) ->
        # options :: {video:boolean,audio:boolean};
        media = new MediaStream()
        success(media)

MediaProxyPeerConnection
========================

      SDP = require 'sdp'
      RTCPeerConnection = require 'rtc-peer-connection'
      request = require 'superagent'

      class MediaProxyPeerConnection extends RTCPeerConnection

In order to build an SDP offer we must first allocate a port on the mediaproxy.

        sdpOffer: (success,failure) ->
          if @_localSession?
            success @_localSession
          media = new MediaStream()

FIXME: This assumes a single mediastream per leg. A generic mediaproxy should handle multiple streams.

          options =
            sip_leg:
              local:
                address: config.media.address
                port: null # dynamically allocated

          request.put "#{config.media.proxy_base}/proxy/#{media.id}", options, (err,res) ->
            leg = res?.body?.sip_leg
            if err? or leg?.error?
              return failure err ? res.body.sip_leg.error

FIXME: This assumes GSM.

            @_media = media
            session = new SDP.Session
              origin:
                username: 'MediaProxy'
                sessionID: media.id
                sessionVersion: 1
                netType: 'IN'
                addrType: 'IP4'
                address: config.signalling.address
              # connection information is shared
              connection:
                netType: 'IN'
                addrType: 'IP4'
                address: leg.local.address
              media: [
                {
                  type: 'audio'
                  port: leg.local.port
                  numberOfPorts: 1
                  proto: 'RTP/AVP' # RFC3551
                  formats: [ 3 ] # actually may be GSM, GSM-EFR, or GSM-HR-08; for now GSM
                  attributes: [ 
                    ## rtpmap is optional since the format (3) is well-known.
                    # { key: 'rtpmap', value: '3 GSM/8000'  }
                    { key: 'ptime', value: 20 }
                  ]
                }
              ]
            success session

        sdpAnswer: (success,failure) ->
          @sdpOffer success, failure

        setLocalSDP: (session,success,failure) ->
          # I don't think we really care here, since all processing was previously done
          # by sdpOffer or sdpAnswer.
          success()

        setRemoteSDP: (session,success,failure) ->
          mediaproxy.get "/proxy/#{@_media.id}", (err,res) ->
            options = res?.body
            if err? or not options?
              failure err

            options.sip_leg ?= {}

FIXME: This assumes a single stream, and hardcode our way through the first audio stream.

            first_audio_media = session.media?.filter( (_) -> _.type is 'audio').shift()
            if first_audio_media?
              options.sip_leg.remote =
                address: session.connection?.address ? first_audio_media.connection?.address
                port: first_audio_media.port

              mediaproxy.put "/proxy/#{media.id}", options, (err,res) ->
                if err?
                  return failure err
                success()

      RTCSessionDescription = require 'rtc-session-description'

      WebRTC =
        getUserMedia: getUserMedia,
        RTCPeerConnection: RTCPeerConnection,
        RTCSessionDescription: RTCSessionDescription
