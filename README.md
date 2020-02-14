# app.rb
require 'line/bot'

class MooncakeController < ApplicationController
    protect_from_forgery with: :null_session
    
    def eat
        render plain: "吃土啦"
    end

    def request_headers
        render plain: request.headers.to_h.reject{|key, value| key.include? '.'}.map{|key, value| "#{key}: #{value}"}.sort.join("\n")
    end

    def request_body
        render plain: request.body
    end

    def response_headers
        response.headers['5566'] = 'QQ'
        render plain: response.headers.to_h.map{|key, value| "#{key}: #{value}"}.sort.join("\n")
    end

    def show_response_body
        puts "===這是設定前的response.body: #{response.body}==="
        render plain: "媽個頭"
        puts "===這是設定後的response.body: #{response.body}==="
    end

    def sent_request
        uri = URI('http://127.0.0.1:3000/mooncake/eat')
        http = Net::HTTP.new(uri.host, uri.port)
        http_request = Net::HTTP::Get.new(uri)
        http_response = http.request(http_request)

        render plain: JSON.pretty_generate({
            request_class: request.class,
            response_class: response.class,
            http_request_class: http_request.class,
            http_response_class: http_response.class
        })
    end
    
    def webhook
        #學說話
        reply_text = learn(channel_id, received_text)

        # 關鍵字回覆
        reply_text = keyword_reply(channel_id, received_text) if reply_text.nil?
        
        # 推齊
        reply_text = echo2(channel_id, received_text) if reply_text.nil?
        
        # 記錄對話
        save_to_received(channel_id, received_text)
        save_to_reply(channel_id, reply_text)

        # 傳送訊息到 line
        response = reply_to_line(reply_text)
        
        # toram 事務
        response = toram(received_text) if received_text == 'toram'
        
        # 回應 200
        head :ok
    end
    
    #toram 事務
    
    def toram(received_text)
        return nil if received_text != 'toram'
        
        # 取得 reply token
        reply_token = params['events'][0]['replyToken']
    
        # 設定回覆訊息
        message = {
          "type": "template",
          "altText": "用手機開哦，電腦不支援",
          "template": {
            "type": "buttons",
            "imageAspectRatio": "rectangle",
            "imageSize": "contain",
            #"thumbnailImageUrl": "圖片網址",
            "imageBackgroundColor": "#a8e8fb",
            "title": "托蘭查詢",
            "text": "練等資訊之類的...",
            
            #"defaultAction": {
            #  "type": "message",
            #  "label": "點到圖片或標題",
            #  "text": "0"
            #},
            
            "actions": [
              {
                "type": "postback",
                "label": "練等資訊",
                "data": "練等"
              },
              {
                "type": "message",
                "label": "點我",
                "text": "曄亂把妹"
              },
              {
                "type": "message",
                "label": "點我",
                "text": "曄夜笙歌"
              }
            ]
          }
        }

        # 傳送訊息
        line.reply_message(reply_token, message)
    end

    # 取得對方說的話
    def received_text
        message = params['events'][0]['message']
        message['text'] unless message.nil?
    end

    # 學講話關鍵字回覆，否則查內建資料系統
    def keyword_reply(channel_id, received_text)
        message = KeywordMapping.where(channel_id: channel_id, keyword: received_text).last&.message
        return message unless message.nil?
        
        message = KeywordMapping.where(keyword: received_text).last
        return message unless message.nil?
        
        search_info(received_text)
    end

    # 內建資料查詢系統
    def search_info(info)
        # 內建資料表
            info_mapping = {
              '月餅' => '別講我壞話哦-0-',
              '緹' => '我師父-0-...安抓?'
            }

        # 查內建資料
        info_mapping[received_text]
    end

    # 學說話 (格式: 月餅記憶模式;關鍵字;回覆)
    def learn(channel_id, received_text)
        # 如果開頭不是 月餅記憶模式; 就跳出
        return nil unless received_text[0..6] == '月餅記憶模式;'
        # 防止 "toram" 被教成關鍵字
        return nil if received_text[7..11] == 'toram'

        received_text = received_text[7..-1]
        semicolon_index = received_text.index(';')

        # 找不到分號就跳出
        return nil if semicolon_index.nil?

        keyword = received_text[0..semicolon_index-1]
        message = received_text[semicolon_index+1..-1]

        KeywordMapping.create(channel_id: channel_id, keyword: keyword, message: message)
        '怕爆'
    end
    
    # 頻道 ID
    def channel_id
        source = params['events'][0]['source']
        source['groupId'] || source['roomId'] || source['userId']
    end
    
    # 儲存對話
    def save_to_received(channel_id, received_text)
        return if received_text.nil?
        Received.create(channel_id: channel_id, text: received_text)
    end
    
    # 儲存回應
    def save_to_reply(channel_id, reply_text)
        return if reply_text.nil?
        Reply.create(channel_id: channel_id, text: reply_text)
    end
    
    def echo2(channel_id, received_text)
        # 如果在 channel_id 最近沒人講過 received_text，月餅就不回應
        recent_received_texts = Received.where(channel_id: channel_id).last(5)&.pluck(:text)
        return nil unless received_text.in? recent_received_texts
    
        # 如果在 channel_id 月餅上一句回應是 received_text，月餅就不回應
        last_reply_text = Reply.where(channel_id: channel_id).last&.text
        return nil if last_reply_text == received_text

        received_text
    end

    # 傳送訊息到 line
    def reply_to_line(reply_text)
        return nil if reply_text.nil?

        # 取得 reply token
        reply_token = params['events'][0]['replyToken']
    
        # 設定回覆訊息
        message = {
          type: 'text',
          text: reply_text
        } 

        # 傳送訊息
        line.reply_message(reply_token, message)
    end

    # Line Bot API 物件初始化
    def line
        @line ||= Line::Bot::Client.new { |config|
          config.channel_secret = 'aaa0acacc70a323c6ec1a98bae71b012'
          config.channel_token = 'put81/Pc6K+sGq4qRMOEc8Q+aRmW9NtJ010uPgc8VVxMuDklXQDnUrb6joPlOCvG+DSUcoiyJ9nT4JaxDjQvAAe8xBniiMCjH4bzvJi+mRnW/kweYNBw46deuVaTIKLVM3MyL2lnu8TSGv3rlX78GQdB04t89/1O/w1cDnyilFU='
        }
    end
    


end    
