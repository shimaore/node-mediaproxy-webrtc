    module.exports = (config) ->

      # Obviously getUserMedia() is useless, since we will have multiple users
      # (and therefor user streams).
      # So we provide a dummy implementation for MediaStream and getUserMedia().

      # http://www.w3.org/TR/mediacapture-streams/#idl-def-MediaStream
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

      #  http://www.w3.org/TR/mediacapture-streams/
      getUserMedia = (options,success,failure) ->
        # options :: {video:boolean,audio:boolean};
        media = new MediaStream()
        success(media)

      # ----------------------------------------------------

      RTCPeerConnection = require 'rtc-peer-connection'

      class MediaProxyPeerConnection extends RTCPeerConnection
        sdpOffer: (success,failure) ->
          if @_media?
            success(@_localSession)
          media = new MediaStream()
          options =
            sip_leg:
              local:
                address: config.local.media.address

          mediaproxy.put "/proxy/#{media.id}", options, (err,res) ->
            if err?
              return failure err

            @_media = media
            session = new SDP.Session
              origin:
                username: JsSIP.name
                sessionID: media.id
                sessionVersion: 1
                netType: 'IN'
                addrType: 'IP4'
                address: config.local.signalling.address
              # connection information is shared
              connection:
                netType: 'IN'
                addrType: 'IP4'
                address: config.local.media.address
              media: [
                {
                  type: 'audio'
                  port: res.local.port
                  numberOfPorts: 1
                  proto: 'RTP/AVP'
                  formats: [ 3 ]
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
            res.sip_leg.remote =
              address: session.connection?.address ? session.media?[0]?.connection?.address
              port: session.media?[0]?.port

            mediaproxy.put "/proxy/#{media.id}", options, (err,res) ->
              if err?
                return failure err
              success()

      RTCSessionDescription = require 'rtc-session-description'

      WebRTC =
        getUserMedia: getUserMedia,
        RTCPeerConnection: RTCPeerConnection,
        RTCSessionDescription: RTCSessionDescription
