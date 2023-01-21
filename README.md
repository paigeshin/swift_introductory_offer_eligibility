# swift_introductory_offer_eligibility

```swift

// Async
func fetch(excludeOldTransaction: Bool, completion: @escaping(Result<[String: Any], Error>) -> Void) {
#if DEBUG
    let urlString: String = "https://sandbox.itunes.apple.com/verifyReceipt"
#else
    let urlString: String = "https://buy.itunes.apple.com/verifyReceipt"
#endif
    
    guard
        let receiptURL: URL = Bundle.main.appStoreReceiptURL,
        let receiptString: String = try? Data(contentsOf: receiptURL).base64EncodedString(),
        let url: URL = URL(string: urlString) else {
        Log.info("Can't create url")
        completion(.success([:]))
        return
    }
    
    let requestData : [String : Any] = ["receipt-data" : receiptString,
                                        "password" : Config.sharedSecrent,
                                        "exclude-old-transactions" : excludeOldTransaction]
    
    let httpBody: Data? = try? JSONSerialization.data(withJSONObject: requestData, options: [])
    var request: URLRequest = URLRequest(url: url)
    request.httpMethod = "POST"
    request.setValue("Application/json", forHTTPHeaderField: "Content-Type")
    request.httpBody = httpBody
    URLSession.shared.dataTask(with: request)  { (data, response, error) in
        if let error: Error = error {
            completion(.failure(error))
            return
        }
        do {
            guard
                let data: Data = data,
                let json: [String: Any] = try JSONSerialization.jsonObject(with: data, options: []) as? [String: Any] else {
                completion(.success([:]))
                return
            }
            completion(.success(json))
        } catch {
            completion(.failure(error))
        }
    }.resume()
    
}



```


```swift

    fetch(excludeOldTransaction: false) { result in
            switch result {
            case .success(let dic):
                Log.info("Receipt Result")
                guard
                    let receipt = dic["receipt"] as? [String: Any],
                    let inApp = receipt["in_app"] as? [[String: Any]]
                else {
                    return
                }
                var latestExpiresDate = Date(timeIntervalSince1970: 0)
                let formatter = DateFormatter()
                for receipt in inApp {
                    let usedTrial : Bool = receipt["is_trial_period"] as? Bool ?? false || (receipt["is_trial_period"] as? NSString)?.boolValue ?? false
                    let usedIntro : Bool = receipt["is_in_intro_offer_period"] as? Bool ?? false || (receipt["is_in_intro_offer_period"] as? NSString)?.boolValue ?? false
                    if usedTrial || usedIntro {
                        //                        callback(false)
                        Log.info("User used trial or intro")
                        return
                    }
                    
                    formatter.dateFormat = "yyyy-MM-dd HH:mm:ss VV"
                    if let expiresDateString = receipt["expires_date"] as? String, let date = formatter.date(from: expiresDateString) {
                        if date > latestExpiresDate {
                            latestExpiresDate = date
                        }
                    }
                    
                    if latestExpiresDate > Date() {
//                      callback(false)
                        Log.info("User is subscribed, not eligible")
                    } else {
                        //                      callback(true)
                        Log.info("User is not subscribed, eligible")
                    }

                    if receipt["cancellation_date"] != nil {
                    // if user made a refund, no need to check for eligibility
//                        callback(false)
                        Log.info("User refunded, not eligible")
                        return
                    }

                    
                }
        
                

            case .failure(let error):
                Log.error(error)
            }
        }

```
