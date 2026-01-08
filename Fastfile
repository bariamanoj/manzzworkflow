default_platform(:ios)

platform :ios do
  before_all do
    setup_ci if ENV['CI']
  end

  desc "Create app on App Store Connect if it doesn't exist"
  lane :create_app do
    begin
      # Use session-based auth for produce (API key not supported)
      ENV["FASTLANE_SESSION"] = ENV["FASTLANE_SESSION_SECRET"]
      ENV["FASTLANE_PASSWORD"] = ENV["FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD_SECRET"]
      
      produce(
        username: "dohrasanket@gmail.com",
        app_identifier: ENV["BUNDLE_IDENTIFIER"],
        app_name: ENV["APP_NAME"],
        language: "en-US",
        app_version: "1.0",
        sku: "#{ENV["BUNDLE_IDENTIFIER"].gsub('.', '-')}-#{Time.now.to_i}",
        skip_itc: false
      )
      UI.success("App created successfully or already exists")
    rescue => ex
      UI.error("App creation failed: #{ex.message}")
      # Continue anyway - app might already exist
    end
  end

  desc "Setup code signing"
  lane :setup_signing do
    api_key = app_store_connect_api_key(
      key_id: ENV["APP_STORE_CONNECT_API_KEY_KEY_ID"],
      issuer_id: ENV["APP_STORE_CONNECT_API_KEY_ISSUER_ID"],
      key_content: ENV["APP_STORE_CONNECT_API_KEY_KEY"]
    )

    create_keychain(
      name: "temp_keychain",
      password: "temp_password",
      default_keychain: true,
      unlock: true,
      timeout: 3600,
      lock_when_sleeps: false
    )

    # Disable readonly mode explicitly
    ENV["MATCH_READONLY"] = "false"

    match(
      type: "appstore",
      app_identifier: ENV["BUNDLE_IDENTIFIER"],
      git_url: "https://github.com/bariamanoj/ios-certificates-new",
      username: "dohrasanket@gmail.com",
      team_id: "42FLQUC3A9",
      keychain_name: "temp_keychain",
      keychain_password: "temp_password",
      api_key: api_key,
      readonly: false
    )
  end

  desc "Build IPA"
  lane :build_ipa do
    # Auto-detect workspace or project
    workspace_path = Dir.glob("*.xcworkspace").first
    project_path = Dir.glob("*.xcodeproj").first
    
    if workspace_path
      build_config = { workspace: workspace_path }
    elsif project_path
      build_config = { project: project_path }
    else
      UI.user_error!("No .xcworkspace or .xcodeproj found")
    end

    gym(
      **build_config,
      scheme: ENV["BUNDLE_IDENTIFIER"].split('.').last,
      export_method: "app-store",
      output_directory: "./build",
      output_name: "app.ipa"
    )
  end

  desc "Upload to TestFlight"
  lane :upload_testflight do
    api_key = app_store_connect_api_key(
      key_id: ENV["APP_STORE_CONNECT_API_KEY_KEY_ID"],
      issuer_id: ENV["APP_STORE_CONNECT_API_KEY_ISSUER_ID"],
      key_content: ENV["APP_STORE_CONNECT_API_KEY_KEY"]
    )

    upload_to_testflight(
      api_key: api_key,
      ipa: "./build/app.ipa",
      skip_waiting_for_build_processing: true
    )
  end

  desc "Upload metadata"
  lane :upload_metadata do
    api_key = app_store_connect_api_key(
      key_id: ENV["APP_STORE_CONNECT_API_KEY_KEY_ID"],
      issuer_id: ENV["APP_STORE_CONNECT_API_KEY_ISSUER_ID"],
      key_content: ENV["APP_STORE_CONNECT_API_KEY_KEY"]
    )

    # Create age rating config
    age_rating_config = {
      "CARTOON_FANTASY_VIOLENCE" => 0,
      "REALISTIC_VIOLENCE" => 0,
      "PROLONGED_GRAPHIC_VIOLENCE" => 0,
      "PROFANITY_CRUDE_HUMOR" => 0,
      "MATURE_SUGGESTIVE" => 0,
      "HORROR" => 0,
      "MEDICAL_TREATMENT_INFO" => 0,
      "ALCOHOL_TOBACCO_DRUGS" => 0,
      "GAMBLING" => 0,
      "SEXUAL_CONTENT_NUDITY" => 0,
      "GRAPHIC_SEXUAL_CONTENT_NUDITY" => 0,
      "UNRESTRICTED_WEB_ACCESS" => 0,
      "GAMBLING_CONTESTS" => 0
    }
    
    File.write("age_rating.json", age_rating_config.to_json)

    deliver(
      api_key: api_key,
      app_identifier: ENV["BUNDLE_IDENTIFIER"],
      skip_binary_upload: true,
      skip_screenshots: true,
      force: true,
      metadata_path: "./fastlane/metadata",
      app_rating_config_path: "./age_rating.json",
      submission_information: {
        export_compliance_uses_encryption: false,
        export_compliance_is_exempt: true,
        export_compliance_contains_third_party_cryptography: false,
        export_compliance_contains_proprietary_cryptography: false,
        export_compliance_available_on_french_store: true
      }
    )
  end

  desc "Setup privacy details"
  lane :setup_privacy do
    api_key = app_store_connect_api_key(
      key_id: ENV["APP_STORE_CONNECT_API_KEY_KEY_ID"],
      issuer_id: ENV["APP_STORE_CONNECT_API_KEY_ISSUER_ID"],
      key_content: ENV["APP_STORE_CONNECT_API_KEY_KEY"]
    )

    privacy_config = {
      "data_protections" => {
        "DEVICE_ID" => {
          "collected" => true,
          "linked" => true,
          "used_for_tracking" => true,
          "purposes" => ["ANALYTICS", "APP_FUNCTIONALITY"]
        },
        "USAGE_DATA" => {
          "collected" => true,
          "linked" => true,
          "used_for_tracking" => true,
          "purposes" => ["ANALYTICS", "APP_FUNCTIONALITY"]
        },
        "ADVERTISING_DATA" => {
          "collected" => true,
          "linked" => true,
          "used_for_tracking" => true,
          "purposes" => ["ANALYTICS", "APP_FUNCTIONALITY"]
        }
      }
    }
    
    File.write("privacy_config.json", privacy_config.to_json)

    upload_app_privacy_details_to_app_store(
      api_key: api_key,
      app_identifier: ENV["BUNDLE_IDENTIFIER"],
      json_path: "./privacy_config.json"
    )
  end

  desc "Set pricing and availability"
  lane :set_pricing do
    require 'jwt'
    require 'net/http'
    require 'json'

    # Generate JWT token
    private_key = OpenSSL::PKey::EC.new(ENV["APP_STORE_CONNECT_API_KEY_KEY"])
    payload = {
      iss: ENV["APP_STORE_CONNECT_API_KEY_ISSUER_ID"],
      exp: Time.now.to_i + 1200,
      aud: "appstoreconnect-v1"
    }
    token = JWT.encode(payload, private_key, "ES256", { kid: ENV["APP_STORE_CONNECT_API_KEY_KEY_ID"] })

    # Get app ID
    uri = URI("https://api.appstoreconnect.apple.com/v1/apps?filter[bundleId]=#{ENV["BUNDLE_IDENTIFIER"]}")
    http = Net::HTTP.new(uri.host, uri.port)
    http.use_ssl = true
    
    request = Net::HTTP::Get.new(uri)
    request['Authorization'] = "Bearer #{token}"
    request['Content-Type'] = 'application/json'
    
    response = http.request(request)
    apps_data = JSON.parse(response.body)
    
    if apps_data['data'] && !apps_data['data'].empty?
      app_id = apps_data['data'][0]['id']
      
      # Set pricing to free (tier 0)
      pricing_uri = URI("https://api.appstoreconnect.apple.com/v1/appPriceSchedules")
      pricing_request = Net::HTTP::Post.new(pricing_uri)
      pricing_request['Authorization'] = "Bearer #{token}"
      pricing_request['Content-Type'] = 'application/json'
      
      pricing_data = {
        data: {
          type: "appPriceSchedules",
          attributes: {
            baseTerritory: "USA",
            manualPrices: [
              {
                territory: "USA",
                startDate: Time.now.strftime("%Y-%m-%d")
              }
            ]
          },
          relationships: {
            app: {
              data: {
                type: "apps",
                id: app_id
              }
            },
            appPricePoint: {
              data: {
                type: "appPricePoints",
                id: "eyJzIjoiVVNBIiwidCI6IjAifQ" # Tier 0 (Free)
              }
            }
          }
        }
      }
      
      pricing_request.body = pricing_data.to_json
      pricing_response = http.request(pricing_request)
      
      if pricing_response.code.to_i >= 200 && pricing_response.code.to_i < 300
        UI.success("Pricing set to free successfully")
      else
        UI.error("Failed to set pricing: #{pricing_response.body}")
      end
    else
      UI.error("App not found for pricing setup")
    end
  rescue => ex
    UI.error("Pricing setup failed: #{ex.message}")
  end

  desc "Full deployment pipeline"
  lane :deploy do
    create_app
    setup_signing
    build_ipa
    upload_testflight
    upload_metadata
    setup_privacy
    set_pricing
  end
end
