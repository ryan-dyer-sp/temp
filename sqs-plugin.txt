module Fluent
    require 'aws-sdk'
    require 'rest-client'
    require 'openssl'
    require 'json'
    require 'date'

    SQS_BATCH_SEND_MAX_MSGS = 10
    SQS_BATCH_SEND_MAX_SIZE = 262144

    RESPONSE_TIMEOUT = 30
    RESTCLIENT_TIMEOUT = 10
    RESTCLIENT_OPENTIMEOUT = 10

    BACKOFF_MAX = 600

  class SQSOutputSPTSTS < Output
    Fluent::Plugin.register_output('sqs_spt_sts', self)

    include SetTagKeyMixin
    config_set_default :include_tag_key, false

    include SetTimeKeyMixin
    config_set_default :include_time_key, true

    config_param :pod, :string
    config_param :org, :string
    config_param :api_user, :string, secret: true
    config_param :api_key, :string, secret: true
    config_param :dev, :bool, default: false
    config_param :queue_name, :string
    config_param :sqs_endpoint, :string
    config_param :proxy, :string, default: ''

    def configure(conf)
      super
    end

    def start
      super

      host = pod + (@dev ? '.cloud.sailpoint.com' : '.accessiq.sailpoint.com')
      url = "https://#{host}/#{org}"

      log.debug("URL: #{url}")

      AWS.config(proxy_uri: @proxy) unless @proxy.empty?

      options = {
        :timeout => RESTCLIENT_TIMEOUT,
        :open_timeout => RESTCLIENT_OPENTIMEOUT,
        :user => @api_user,
        :password => @api_key,
        :verify_ssl => OpenSSL::SSL::VERIFY_PEER,
        :headers => {
          "X-CSRF-Token" => "nocheck"
        }
      }

      # set proxy settings if present
      RestClient.proxy = @proxy unless @proxy.empty?

      @api = RestClient::Resource.new(url, options)

      @sleep = 1

      @expire = Time.now
    end

    def shutdown
      super
    end

    def emit(tag, es, chain)
      chain.next

      batch_records = []
      batch_size = 0
      send_batches = [batch_records]

      es.each do |time,record|
        body = JSON.dump(record)
        log.info("Body: #{body}")
        batch_size += body.bytesize

        if batch_size > SQS_BATCH_SEND_MAX_SIZE ||
          batch_records.length >= SQS_BATCH_SEND_MAX_MSGS

          batch_records = []
          batch_size = body.bytesize
          send_batches << batch_records
        end

        if batch_size > SQS_BATCH_SEND_MAX_SIZE then
          log.warn "Could not push message to SQS, payload exceeds "\
            "#{SQS_BATCH_SEND_MAX_SIZE} bytes.  "\
            "(Truncated message: #{body[0..200]})"
        else
          batch_records << {
            message_body: body,
            delay_seconds: 0
          }
        end

        send_batches.each do |records|
          begin
            log.info("Records: #{records}")
            get_aws.batch_send(records) unless records.empty?
            log.info("Records sent")
          rescue Exception => e
            log.error("Error during SQS operation: #{e}")
            log.error("Records: #{records}")
            log.info("Waiting #{@sleep} seconds before retry...")
            sleep @sleep
            @sleep *= 2
            @sleep = BACKOFF_MAX if @sleep > BACKOFF_MAX
            retry
          end
        end

      end
    end

    private

    def get_aws
      if instance_variable_defined?('@sqs') && Time.now < (@expire - 10)
        @sqs
      else
        log.info("New log key requested at: #{Time.now}")
        request_credentials
        @sqs
      end
    end

    def request_credentials
      content = {content_type: :json, accept: :json}

      out = JSON.parse(@api['/api/client/requestCredentials'].post(content))

      @sqs = AWS::SQS.new(sqs_endpoint: @sqs_endpoint,
                          access_key_id: out['aws.accessKeyId'],
                          secret_access_key: out['aws.secretAccessKey'],
                          session_token: out['aws.sessionToken']
                         ).queues.named(@queue_name)

      @expire = DateTime.parse(out['aws.expiration']).to_time

    end

  end
end