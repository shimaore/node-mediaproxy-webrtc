MediaProxyStream(Track)
=======================

See http://www.w3.org/TR/mediacapture-streams/#idl-def-MediaStream

Note: this probably is a `MediaStreamTrack` in the new WebRTC parlance.
Also this is a two-way stream, so it serves both purposes of a `sending` (localStream) and `receiving` (remoteStream) wrt RTCPeerConnection.

    SDP = require 'sdp'
    request = require 'superagent'

    class MediaProxyMediaStream

      ### Typical @constraints value:
        media:
          address: # IP address of mediaproxy Media
          proxy_base: # base URI for access to MediaProxy
        description:
          type: 'audio'
          formats: [ 3 ] # actually may be GSM, GSM-EFR, or GSM-HR-08; for now GSM
          attributes: [
            ## rtpmap is optional since the format (3) is well-known.
            # { key: 'rtpmap', value: '3 GSM/8000'  }
            { key: 'ptime', value: 20 }
          ]
      ###

      constructor: (@constraints) ->
        @id = new uuid();

        # @onended = null;
        # @onaddtrack = null;
        # @onremovetrack = null;

A sending stream must provide the equivalent of `getUserMedia`:
For `getUserMedia` see http://www.w3.org/TR/mediacapture-streams/

      get: (success,failure) ->

        options =
          sip_leg:
            local:
              address: constraints.media.address
              port: null # dynamically allocated

        request.put "#{@constraints.media.proxy_base}/proxy/#{@id}", options, (err,res) =>
          if err? then return failure err

          leg = res?.body?.sip_leg
          if not leg? then return failure 'Unable to build leg'
          if leg.error? then return failure leg.error

          @description.port = leg.local.port
          @description.numberOfPorts = 1
          @description.proto = 'RTP/AVP' # RFC3551
          @description.connection =
            netType: 'IN'
            addrType: 'IP4'
            address: leg.local.address

          success this

A receiving stream must provide the equivalent of `setRemoteDescription`:

      setRemoteSession: (session,success,failure) ->
        mediaproxy.get "#{@constraints.media.proxy_base}/proxy/#{@id}", (err,res) =>
          if err? then return failure err

          options = res?.body
          if not options? then return failure 'Could not set options'

          options.sip_leg ?= {}
          options.sip_leg.remote =
            address: session.connection?.address ? @description.connection?.address
            port: first_audio_media.port

          mediaproxy.put "#{@constraints.media.proxy_base}/proxy/#{@id}", options, (err,res) =>
              if err?
                return failure err
              success()

      stop: ->
        request.delete "#{@constraints.media.proxy_base}/proxy/#{@id}", options, (err,res) =>
          if err?
            throw new Error err

      #  MediaStream.prototype.getAudioTracks
      #  MediaStream.prototype.getVideoTracks
      #  etc.: not implemented
      #  Doesn't look like we're using any other attribute.

    module.exports =
      getUserMedia: (constraints,success,failure) ->
        media = new MediaProxyMediaStream constraints
        media.get success, failure
      RTCPeerConnection: require 'rtc-peer-connection'
      RTCSessionDescription: require 'rtc-session-description'
      isSupported: true
