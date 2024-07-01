package de.icx.conn.whatsapphub;

import java.io.File;
import java.io.IOException;
import java.util.ArrayList;
import java.util.Base64;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Properties;

import javax.servlet.ServletContextEvent;
import javax.servlet.ServletContextListener;
import javax.xml.bind.annotation.XmlRootElement;

import org.apache.http.client.fluent.Request;
import org.apache.log4j.Logger;
import org.apache.log4j.MDC;

import de.icx.common.Http4;
import de.icx.common.Http4.ResponseInfo;
import de.icx.common.Json;
import de.icx.common.Prop;
import de.icx.common.basics.CList;

// import com.genesyslab.mcr.smserver.gframework.CfgOptions;

import de.icx.common.basics.CMap;
import de.icx.common.basics.Common;
import de.icx.conn.whatsapphub.objects.Token;

// TODO: Use Genesys configuration?

public class WCL extends Common implements ServletContextListener {

	protected static Logger log = Logger.getLogger(WCL.class);

	public final static String WHATSAPP_PROPERTY_FILE = "whatsapp.properties";

	public final static String AUTHORIZATION = "authorization";
	public final static String BASIC = "Basic "; // Space!
	public final static String BEARER = "Bearer "; // Space!

	@XmlRootElement
	public static class Bearer {
		public String token = null;
		public String expires_after = null;
	}

	public static File tomcatDir = null;
	public static File servletDir = null;

	public static String environment;

	// Configured business phones for configured environment
	public static List<String> businessPhones = new ArrayList<>();

	public static String fourthPlatformUrl = null;

	public static int acceptedObjectsInQueue = 1;

	public static Map<String, Map<String, String>> basicAuthHeaderMapByBusinessPhoneMap = new HashMap<>();
	public static Map<String, Map<String, String>> bearerAuthHeaderMapByBusinessPhoneMap = new HashMap<>();

	static Map<String, Thread> whatsappSessionManagementThreadByBusinessPhoneMap = new HashMap<>();
	private static boolean whatsappSessionManagementIsRunning = false;

	// Daily login and register webhooks URL (Whatsapp session runs out after 7 days)
	public class FourthPlatformSessionManagement implements Runnable {

		String businessPhone = null;

		public FourthPlatformSessionManagement(
				String businessPhone) {
			this.businessPhone = businessPhone;
		}

		@Override
		public void run() {

			MDC.put("THREAD", "4P_SESSION_MANAGEMENT");

			log.info("WCL: Whatsapp session management thread for '" + businessPhone + "' started");

			Map<String, String> basicAuthHeaderMap = basicAuthHeaderMapByBusinessPhoneMap.get(businessPhone);

			// Login to Whatsapp and register webhooks URL (Whatsapp notifications listener)
			try {
				Thread.sleep(10000);
			}
			catch (InterruptedException e1) {
			}

			ResponseInfo ri = null;
			while (whatsappSessionManagementIsRunning) {

				try {
					// Try to receive access token...
					boolean accessTokenReceived = false;
					while (!accessTokenReceived) {

						// Retrieve access token (4p specific)
						ri = Http4.executeAndLog(Request.Post(fourthPlatformUrl + "/token"), basicAuthHeaderMap, CMap.newMap("grant_type", "client_credentials"), log, "WCL:");
						if (ri.wasSuccessful()) {
							accessTokenReceived = true;
						}
						else {
							log.error("WCL: Retrieving access token for '" + businessPhone + "' unsuccessful!");
							Thread.sleep(3000);
						}
					}

					// Get bearer code for subsequent bearer authorization and build bearer authorization header used in all requests within this Whatsapp session
					@SuppressWarnings("null")
					Token token = Json.unmarshal(Token.class, ri.content, false, log);
					log.info("WCL: Token '" + token.access_token + "' received for '" + businessPhone + "'");
					bearerAuthHeaderMapByBusinessPhoneMap.put(businessPhone, CMap.newStringStringMap(AUTHORIZATION, BEARER + token.access_token, "x-correlator", "WA4P-hub"));

					// Sleep until session times out to force session refresh
					int sleepTime = token.expires_in - 60;
					log.info("WCL: Sleep " + sleepTime + " seconds until session refresh (for '" + businessPhone + "') ...");
					Thread.sleep(sleepTime * 1000);
				}
				catch (InterruptedException e) {
					log.info("WCL: Session management thread for '" + businessPhone + "' interrupted");
					if (whatsappSessionManagementIsRunning) {
						log.warn("WCL: Bearer session seems to be invalid! Get new bearer token and renew session.");
					}
				}
				catch (Exception ex) {
					log.error("WCL: Session management thread for '" + businessPhone + "': Exception " + ex + " occurred!");
				}
			}

			log.info("WCL: Whatsapp session management thread for '" + businessPhone + "' ended");

			MDC.clear();
		}
	}

	public final void startWhathappSessionManagementThread(String businessPhone) {

		whatsappSessionManagementIsRunning = true;

		Thread thread = new Thread(new FourthPlatformSessionManagement(businessPhone));
		whatsappSessionManagementThreadByBusinessPhoneMap.put(businessPhone, thread);

		thread.setName("WhatsappSessionManagement-" + businessPhone + "-" + thread.getId());
		thread.start();
	}

	public final static void forceRenewBearerSession(String businessPhone) {

		Thread thread = whatsappSessionManagementThreadByBusinessPhoneMap.get(businessPhone);
		if (thread != null) {
			thread.interrupt();
		}
	}

	public final static void stopWhathappSessionManagement() {

		for (String businessPhone : businessPhones) {

			Thread thread = whatsappSessionManagementThreadByBusinessPhoneMap.get(businessPhone);
			if (thread != null) {

				whatsappSessionManagementIsRunning = false;
				thread.interrupt();
				try {
					thread.join();
					log.info("WCL: Whatsapp session management thread for '" + businessPhone + "' terminated");
				}
				catch (InterruptedException inex) {
					log.error("WCL: " + Common.exceptionStackToString(inex));
				}
				thread = null;
			}
		}
	}

	public static File getTomcatConfDirForServlet() {

		if (tomcatDir != null) {
			return new File(tomcatDir, "conf/" + Common.untilFirst(servletDir.getName(), "#"));
		}
		else {
			return new File("C:/tmp");
		}
	}

	@Override
	public void contextInitialized(ServletContextEvent event) {

		MDC.put("THREAD", "WHATSAPP_HUB");

		try {
			// Get directories
			tomcatDir = new File(event.getServletContext().getRealPath("."), "../..").getCanonicalFile();
			servletDir = new File(event.getServletContext().getRealPath(".")).getCanonicalFile();

			// Environment
			environment = event.getServletContext().getInitParameter("environment");
			if (environment != null) {
				log.info("WCL: Current environment '" + environment + "' pre-configured as context parameter 'environment' in 'web.xml'");
			}
			else {
				log.info("WCL: Current environment not configured as context parameter 'environment' in 'web.xml' - check '" + WHATSAPP_PROPERTY_FILE + "' file for current environment...");
			}

			// Find and read Whatsapp properties file - first try to find it in servlet specific subdirectory of <TOMCAT>/conf dir
			File whatsappPropertiesFile = new File(getTomcatConfDirForServlet(), WHATSAPP_PROPERTY_FILE);
			if (!whatsappPropertiesFile.canRead()) {
				log.warn("WCL: Configuration file '" + WHATSAPP_PROPERTY_FILE + "' not found in '" + getTomcatConfDirForServlet() + "'. Try to find it in classpath.");

				// Second try to find property files in classpath (normally directly under 'classes' dir)
				whatsappPropertiesFile = Prop.findPropertiesFile(WHATSAPP_PROPERTY_FILE);
				if (whatsappPropertiesFile == null) {
					log.error("WCL: No or multiple properties files '" + WHATSAPP_PROPERTY_FILE + "' found in classpath! Cannot read necessary properties.");
					return;
				}
			}
			else {
				log.info("WCL: Use configuration file '" + WHATSAPP_PROPERTY_FILE + "' in '" + getTomcatConfDirForServlet() + "'");
			}

			// Try to get current environment from 'whatsapp.properties'
			Properties allProperties = Prop.readProperties(whatsappPropertiesFile);
			if (allProperties.containsKey("environment")) {
				environment = allProperties.getProperty("environment");
				log.info("WCL: Current environment '" + environment + "' directly configured in '" + WHATSAPP_PROPERTY_FILE + "'");
			}
			if (isEmpty(environment)) {
				log.error("WCL: Current environment neither configured as context parameter 'environment' in 'web.xml' nor directly as property 'environment' in '" + WHATSAPP_PROPERTY_FILE + "'!");
				return;
			}

			// Get environment specific properties
			Properties environmentSpecificProperties = Prop.readEnvironmentSpecificProperties(whatsappPropertiesFile, environment, CList.newList("businessPhones", "fourthPlatformUrl"));

			// Get 4P URL
			fourthPlatformUrl = environmentSpecificProperties.getProperty("fourthPlatformUrl");
			if (isEmpty(fourthPlatformUrl)) {
				log.error("WCL: Property 'fourthPlatformUrl' could not be found in '" + WHATSAPP_PROPERTY_FILE + "'!");
				return;
			}
			else {
				log.info("WCL: 4P URL for environment '" + environment + "': " + fourthPlatformUrl);
			}

			// Get business phones for given environment
			String businessPhonesString = environmentSpecificProperties.getProperty("businessPhones");
			if (isEmpty(businessPhonesString) || stringToList(businessPhonesString).isEmpty()) {
				log.error("WCL: One or more business account phone numbers (e.g. 4991147875096) must be configured for current ennvironment '" + environment
						+ "' as comma separated list in property 'businessPhones' in '" + WHATSAPP_PROPERTY_FILE + "'!");
				return;
			}
			else {
				for (String businessPhone : stringToList(businessPhonesString)) {
					businessPhones.add(businessPhone);
				}
				log.info("WCL: Business phones for environment '" + environment + "': " + businessPhones);
			}

			// Get business phone specific properties
			for (String businessPhone : businessPhones) {

				Properties businessNumberSpecificProperties = Prop.readEnvironmentSpecificProperties(whatsappPropertiesFile, environment + "/" + businessPhone,
						CList.newList("whatsappUser", "whatsappPwd"));

				// Basic authentication credentials
				String whatsappUser = businessNumberSpecificProperties.getProperty("whatsappUser");
				String whatsappPwd = businessNumberSpecificProperties.getProperty("whatsappPwd");
				if (isEmpty(whatsappUser) || isEmpty(whatsappPwd)) {
					log.error("WCL: Credentials for 4P login for current environment '" + environment + "' and '" + businessPhone + "' could not be found in '" + WHATSAPP_PROPERTY_FILE
							+ "'! ('whatsappUser', 'whatsappPwd')");
					return;
				}
				else {
					log.info("WCL: Credentials for environment '" + environment + "' and business phone '" + businessPhone + "': " + whatsappUser + "/*****");
				}

				// Get incoming objects queue length property
				acceptedObjectsInQueue = Prop.getIntProperty(environmentSpecificProperties, "acceptedObjectsInQueue", 1);
				if (acceptedObjectsInQueue < 1) {
					acceptedObjectsInQueue = 1;
				}
				else if (acceptedObjectsInQueue > 100) {
					acceptedObjectsInQueue = 100;
				}

				// Build authorization header for basic authentication
				basicAuthHeaderMapByBusinessPhoneMap.put(businessPhone, CMap.newMap(AUTHORIZATION, BASIC + Base64.getEncoder().encodeToString((whatsappUser + ":" + whatsappPwd).getBytes())));

				// Initially load messages/errors/status' which were received but not retrieved by DMS (due to DMS downtime)
				Storage.loadObjectsFromDumpFileIfExists(de.icx.conn.whatsapphub.objects.Message.class, businessPhone);
				Storage.loadObjectsFromDumpFileIfExists(de.icx.conn.whatsapphub.objects.Error.class, businessPhone);
				Storage.loadObjectsFromDumpFileIfExists(de.icx.conn.whatsapphub.objects.Status.class, businessPhone);

				// Start thread for cyclic session refresh
				startWhathappSessionManagementThread(businessPhone);
			}
		}
		catch (IOException e) {
			log.error("WCL: Exception occurred trying to read properties file '" + WHATSAPP_PROPERTY_FILE + "':\n" + exceptionStackToString(e));
			return;
		}
	}

	@Override
	public void contextDestroyed(ServletContextEvent event) {

		stopWhathappSessionManagement();

		MDC.clear();
	}

}
