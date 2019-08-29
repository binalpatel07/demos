# demosmport UIKit
import Foundation
import SystemConfiguration
import Alamofire
import SwiftyJSON
//let reachability = Reachability()!
class WebServices: NSObject
{
    var operationQueue = OperationQueue()
    // Call with Raw/Json Parameter
    func CallGlobalAPI(url:String, headers:HTTPHeaders,parameters:NSDictionary, HttpMethod:Alamofire.HTTPMethod, ProgressView:Bool, NetworkAlert:Bool, responseDict:@escaping ( _ jsonResponce:JSON?, _ strErrorMessage:String ) -> Void )  {
        print("URL: \(url)")
        print("Headers: \n\(headers)")
        print("Parameters: \n\(parameters)")
        
        let operation = BlockOperation.init {
            DispatchQueue.global(qos: .background).async {

                    var req = URLRequest(url: try! url.asURL())
                    req.httpMethod = HttpMethod.rawValue
                    req.allHTTPHeaderFields = headers
                    req.setValue("application/json", forHTTPHeaderField: "content-type")
                    req.httpBody = try! JSONSerialization.data(withJSONObject: parameters)
                    req.timeoutInterval = 30 // 10 secs
                    Alamofire.request(req).responseJSON { response in
                        switch (response.result)
                        {
                        case .success:
                            if((response.result.value) != nil) {
                                let jsonResponce = JSON(response.result.value!)
                                print("Responce: \n\(jsonResponce)")
                                DispatchQueue.main.async {
                                    
                                    responseDict(jsonResponce,"")
                                }
                            }
                            break
                        case .failure(let error):
                            let message : String
                            if let httpStatusCode = response.response?.statusCode {
                                switch(httpStatusCode) {
                                case 400:
                                    message = "Something Went Wrong..Try Again"
                                case 401:
                                    message = "Something Went Wrong..Try Again"
                                    DispatchQueue.main.async {
                                        
                                        responseDict([:],message)
                                    }
                                default: break
                                }
                            } else {
                                message = error.localizedDescription
                                let jsonError = JSON(response.result.error!)
                                DispatchQueue.main.async {
                                    
                                    responseDict(jsonError,"")
                                }
                            }
                            break
                        }
                    }
            }
        }
        operation.queuePriority = .normal
        operationQueue.addOperation(operation)
    }
    
}

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////


func Webservice_Login(url: String, params: NSDictionary) -> Void {
    
    WebServices().CallGlobalAPI(url: url, headers: [:], parameters: params, HttpMethod: "POST", ProgressView: true, NetworkAlert: true) {( _ jsonResponse:JSON? , _ strErrorMessage:String) in
        
        if strErrorMessage.count != 0 {
            Loaf(strErrorMessage, state: .error, sender: self).show()
        }
        else {
            print(jsonResponse!)
            let responseStatus = jsonResponse!["status"].stringValue
            if responseStatus == "1" {
                let responseData = jsonResponse!["data"].dictionaryValue
                let loginToken = responseData["token"]?.stringValue
                UserDefaultManager.setStringToUserDefaults(value: loginToken!, key: UD_LoginToken)
                let objVC = self.storyboard?.instantiateViewController(withIdentifier: "TabBarVC") as! TabBarVC
                let nav = UINavigationController(rootViewController: objVC)
                nav.navigationBar.isHidden = true
                let appdelegate = UIApplication.shared.delegate as! AppDelegate
                appdelegate.window!.rootViewController = nav
            }
            else {
                let responseMessage = jsonResponse!["message"].stringValue
                Loaf(responseMessage, state: .error, sender: self).show()
            }
        }
    }
}

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////


    func multipartWebService(method: HTTPMethod, URLString: String,encoding: Alamofire.ParameterEncoding, parameters: [String: Any], fileData: Data!,fileUrl:URL?, headers: [String: String]?,keyName:String, completion: @escaping (_ response:AnyObject?, _ error: NSError?) -> ()){
        print("Fetching WS : \(URLString)")
        print("With parameters : \(parameters)")
        
        if  !NetworkReachabilityManager()!.isReachable {
            //completion(nil, NSError(coder: ErrorCode.networkUnavailable))
        }
        
        Alamofire.upload(multipartFormData: { (multipartFormData) in
            for (key, value) in parameters {
                
                multipartFormData.append("\(value)".data(using: String.Encoding.utf8)!, withName: key as String)
            }
            
            let name = randomString(length: 5)
            
            if let data = fileData {
                multipartFormData.append(data, withName: keyName, fileName: "image\(name).jpeg", mimeType: "image/jpeg")
            }
            
        }, to: URLString, headers: headers) { (encodingResult) in
            //SVProgressHUD.dismiss()
            switch encodingResult {
                
            case .success(request: let upload, streamingFromDisk: _, streamFileURL: _):
                upload.responseJSON { (response) -> Void in
                    if let statusCode = response.response?.statusCode {
                        if  statusCode == HttpResponseStatusCode.noAuthorization.rawValue {
                            //completion(nil, NSError(coder: ErrorCode.invalidCredentials))
                            return
                        }
                    }
                    if let error = response.result.error {
                        completion(nil, error as NSError?)
                    }
                    else {
                        guard let data = response.data
                            else {
                                //completion(nil, NSError(errorCode: ErrorCode.noDataReceived))
                                return
                        }
                        
                        do {
                            let unparsedObject = try JSONSerialization.jsonObject(with: data, options: JSONSerialization.ReadingOptions.allowFragments) as AnyObject
                            //if let responseJson: AnyObject = unparsedObject  {
                            completion(unparsedObject, nil)
                            //}
                        }
                        catch let exception as NSError {
                            completion(nil, exception)
                        }
                    }
                }
            case .failure( _): break
            }
        }
    }
    
    //MARK: Internet Avilability
    func internetChecker(reachability: Reachability) -> Bool {
        var check:Bool = false
        if reachability.connection == .wifi {
            check = true
        }
        else if reachability.connection == .cellular {
            check = true
        }
        else
        {
            check = false
        }
        return check
    }
    
    // MARK: ProgressView
    func ProgressViewShow() {
        DispatchQueue.main.async {
            SVProgressHUD.setDefaultMaskType(.black)
            SVProgressHUD.show()
        }
    }
    
    func ProgressViewHide() {
        DispatchQueue.main.async {
            SVProgressHUD.dismiss()
        }
    }
    
    
    
import UIKit 
import Alamofire
import SwiftyJSON

let BaseURL = WebserviceURLs.kBaseURL
let baseTaskURL = WebserviceURLs.kBaseURL
var request : Request!



//-------------------------------------------------------------
// MARK: - Webservice For PostData Method
//-------------------------------------------------------------

class Connectivity {
    class func isConnectedToInternet() ->Bool {
        return NetworkReachabilityManager()!.isReachable
    }
}

func postData(_ dictParams: AnyObject, nsURL: String, completion: @escaping (_ result: AnyObject, _ sucess: Int) -> Void)
{
    let url = WebserviceURLs.kBaseURL + nsURL
    print("url = \(url) params = \(dictParams)")
    
    let header: [String:String] = ["Content-Type":"application/json"]
    
    Alamofire.request(url, method: .post, parameters: dictParams as? [String : AnyObject], encoding: JSONEncoding.default, headers: header)
        .validate()
        .responseJSON
        { (response) in
            
            if let responsedata = response.data as? Data {
                do {
                    if let jsonArray = try JSONSerialization.jsonObject(with: responsedata, options : .allowFragments) as? [String:Any]
                    {
                        print(jsonArray) // use the json here
                    } else {
                        print("bad json")
                    }
                } catch let error as NSError {
                    print(error)
                }
            }
            if((response.result.value != nil)){
                switch response.result {
                  
                    case .success( _):
                    if let JSON = response.result.value
                    {
                    if (JSON as AnyObject).object(forKey:("status")) as? Int == 0 || (JSON as AnyObject).object(forKey:("success")) as? Int == 0
                    {
                        completion(JSON as AnyObject, 0)
                    }
                      else  if (JSON as AnyObject).object(forKey:("status")) as? Int == 1 || (JSON as AnyObject).object(forKey:("success")) as? Int == 1
                        {
                            completion(response.result.value as AnyObject, 1)
                        }
                       else if (JSON as AnyObject).object(forKey:("status")) as? Int == 2 || (JSON as AnyObject).object(forKey:("success")) as? Int == 2
                        {
                            completion(response.result.value as AnyObject, 2)
                        }
                    }
                    else {
                    completion(response.result.value as AnyObject, 0)
                    
                    }
                    case .failure(_): break
                    
                    }
            }
    }
}

func getData(_ dictParams: AnyObject, nsURL: String,  completion: @escaping (_ result: AnyObject, _ success: Int) -> Void)
{
    let url =  nsURL//WebserviceURLs.kBaseURL + nsURL
    print(url)

    Alamofire.request(url, method: .get, parameters: dictParams as? [String : AnyObject], encoding: URLEncoding.default)
        .validate()
        .responseJSON
        { (response) in
            
            if let JSON = response.result.value
            {
                
                if (JSON as AnyObject).object(forKey:("status")) as! Int == 1
                {
                    completion(JSON as AnyObject, 1)
                }
                else
                {
                    completion(JSON as AnyObject, 0)
                    
                }
            }
            else
            {
                print("Data not Found")
            }

    }
}


    //-------------------------------------------------------------
    // MARK: - Webservice For Send Image Method
    //-------------------------------------------------------------
    
    func sendImage(_ dictParams: AnyObject, image1: UIImage, nsURL: String, completion: @escaping (_ result: AnyObject, _ success: Bool) -> Void) {
        
        let url = WebserviceURLs.kBaseURL + nsURL
        
        let header: [String:String] = ["Content-Type":"application/json"]
        
        let dictData = dictParams as! [String:AnyObject]
        Alamofire.upload(multipartFormData: { (multipartFormData) in
            
            if let imageData1 = UIImage.jpegData(image1)(compressionQuality: 0.3) {
                
                multipartFormData.append(imageData1, withName: "image", fileName: "image.jpeg", mimeType: "image/jpeg")
            }
            
            for (key, value) in dictData
            {
                
                print(value)
                multipartFormData.append(String(describing: value).data(using: .utf8)!, withName: key)
            }
        }, usingThreshold: 10 * 1024 * 1024, to: url, method: .post, headers: header) { (encodingResult) in
            switch encodingResult
            {
            case .success(let upload, _, _):
                request =  upload.responseJSON {
                    response in
                    
                    if let JSON = response.result.value {
                        
                        if ((JSON as AnyObject).object(forKey: "status") as! Bool) == true
                        {
                            completion(response.data as AnyObject, true)
                            print("If JSON")
                            
                        }
                        else
                        {
                            completion(JSON as AnyObject, false)
                            print("else JSON")
                        }
                    }
                    else
                    {
                        print("ERROR")
                    }
                    
                    
                }
            case .failure( _):
                print("failure")
                
                break
            }
        }
    }

//
//  SKROLLIE
//
//  Created by Dhaval Bhanderi on 4/16/19.
//  Copyright © 2019 Dhaval Bhanderi. All rights reserved.
//
import Foundation
import UIKit

let kAPPVesion = Bundle.main.object(forInfoDictionaryKey: "CFBundleShortVersionString") as! String

//-------------------------------------------------------------
// MARK: - ProfileData satatic key
//-------------------------------------------------------------
struct login
{
    static let kProfileData = "profile"
}

//-------------------------------------------------------------
// MARK: - WebserviceURLs
//-------------------------------------------------------------
struct WebserviceURLs {
    
    static let kBaseURL = "http://192.168.0.151/AnTim/medicoster/"
    static let kBaseImageURL = ""
    
    //POST
    
    static let klogin = "login"
    static let kRegister = "signUp"
    static let kHome = "userHome"
    static let kForgot = "forgotpassword"
    static let kmedicineDivisionList = "medicineDivisionList"
    static let kmedicine = "medicine"
    
    //GET
    
    static let kMedicineComList = "http://192.168.0.151/AnTim/medicoster/medicineCompanyList"
    
}
    struct keyAllKey
    {
        
        //Login
        static let MobileNum = "mobileNo"
        static let Password = "password"
        
        //SignUp
        static let MedicalName  = "medicalName"
        static let PersonName  = "personName"
        static let DrugLicenseNo  = "drugLicenseNo"
        static let FoodLicenseNo  = "foodLicenseNo"
        static let GstNo  = "gstNo"
        static let Address  = "address"
        static let Pincode  = "pincode"
        static let Email  = "email"
        static let id = "id"
        
        static let companyId =  "companyId"
    
}
//
//  SKROLLIE
//
//  Created by Dhaval Bhanderi on 4/16/19.
//  Copyright © 2019 Dhaval Bhanderi. All rights reserved.
//

import UIKit
import Alamofire
import SwiftyJSON

//POST METHOD
let Login = WebserviceURLs.klogin
let Register = WebserviceURLs.kRegister
let Forgot = WebserviceURLs.kForgot
let Home = WebserviceURLs.kHome
let MedicineDiv = WebserviceURLs.kmedicineDivisionList
let Medicine =  WebserviceURLs.kmedicine
let Comlist = WebserviceURLs.kMedicineComList

//-------------------------------------------------------------
// MARK: - Webservice For Login
//-------------------------------------------------------------

func webserviceForLogin(_ dictParams: AnyObject, completion: @escaping(_ result: AnyObject, _ success: Int) -> Void)
{
    let url = Login
    print(url)
    postData(dictParams, nsURL: url, completion: completion)
}

func WebserviceofUserHome(_ dictParams: AnyObject, completion: @escaping(_ result: AnyObject, _ success: Int) -> Void)
{
    let url = Home
    print(url)
    postData(dictParams, nsURL: url, completion: completion)
}
func WebserviceofForgot(_ dictParams: AnyObject, completion: @escaping(_ result: AnyObject, _ success: Int) -> Void)
{
    let url = Forgot
    print(url)
    postData(dictParams, nsURL: url, completion: completion)
}
func WebserviceofRegister(_ dictParams: AnyObject, completion: @escaping(_ result: AnyObject, _ success: Int) -> Void)
{
    let url = Register
    print(url)
    postData(dictParams, nsURL: url, completion: completion)
}
func WebserviceofMedidivisiones(_ dictParams: AnyObject, completion: @escaping(_ result: AnyObject, _ success: Int) -> Void)
{
    let url = MedicineDiv
    print(url)
    postData(dictParams, nsURL: url, completion: completion)
}
func WebserviceofMedicine(_ dictParams: AnyObject, completion: @escaping(_ result: AnyObject, _ success: Int) -> Void)
{
    
    let url = Medicine
    print(url)
    postData(dictParams, nsURL: url, completion: completion)
}

//------------------------------------------------------------
// MARK: - Webservice For Comlist Get
//------------------------------------------------------------
func webserviceForComlist(dictParams: AnyObject,completion: @escaping(_ result: AnyObject, _ success: Int) -> Void)
{
    let url = Comlist //+ "\(dictParams)"
    print(url)
    getData(dictParams, nsURL: url, completion: completion)
}

//-------------------------------------------------------------
// MARK: - Webservice For Save profile
//-------------------------------------------------------------
//func webserviceOfSaveProfile(_ dictParams: AnyObject, image1: UIImage, completion: @escaping(_ result: AnyObject, _ success: Bool) -> Void)
//{
//    let url = SaveProfile
//    sendImage(dictParams, image1: image1, nsURL: url, completion: completion)
//}



