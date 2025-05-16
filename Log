package com.rbs.bdd.application.service;


import com.rbs.bdd.application.exception.SchemaValidationException;
import com.rbs.bdd.application.port.out.AccountValidationPort;
import com.rbs.bdd.generated.*;
import lombok.RequiredArgsConstructor;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import org.springframework.ws.WebServiceMessage;
import org.springframework.ws.soap.saaj.SaajSoapMessage;
import org.w3c.dom.Document;
import org.w3c.dom.Node;
import javax.xml.parsers.DocumentBuilder;
import javax.xml.parsers.DocumentBuilderFactory;
import javax.xml.transform.Transformer;
import javax.xml.transform.TransformerFactory;
import javax.xml.transform.dom.DOMSource;
import javax.xml.transform.stream.StreamResult;
import javax.xml.transform.stream.StreamSource;
import javax.xml.xpath.XPath;
import javax.xml.xpath.XPathConstants;
import javax.xml.xpath.XPathFactory;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.InputStream;
import java.lang.Exception;


/**
 * Service class responsible for validating SOAP request schema and applying business rules to static XML response.
 */
@Service
@RequiredArgsConstructor
public class AccountValidationService implements AccountValidationPort {

    private static final Logger logger = LoggerFactory.getLogger(AccountValidationService.class);


    /**
     * Automatically validates the request schema via Spring WS's interceptor.
     */
    @Override
    public void validateSchema(ValidateArrangementForPaymentRequest request) {
        logger.info("Schema validation completed (Spring WS handles XSD check)");
    }

    /**
     * Applies business rules to a static SOAP response XML file, modifies specific fields using DOM/XPath,
     * and writes the transformed XML directly to the outgoing SOAP message.
     *
     * @param request SOAP request containing operating brand
     * @param message Outgoing SOAP response to be written with modified static content
     */
        @Override
        public void validateBusinessRules(ValidateArrangementForPaymentRequest request, WebServiceMessage message) {
            try {
                logger.info("Starting business rule processing for operating brand: {}", request.getRequestHeader().getOperatingBrand());

                InputStream xml = getClass().getClassLoader().getResourceAsStream("static-response/response1.xml");
                if (xml == null) {
                    throw new SchemaValidationException("response1.xml not found in classpath");
                }

                DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
                factory.setNamespaceAware(true);
                DocumentBuilder builder = factory.newDocumentBuilder();
                Document doc = builder.parse(xml);

                XPath xpath = XPathFactory.newInstance().newXPath();

                // Update systemId
                Node systemId = (Node) xpath.evaluate(
                        "//*[local-name()='responseId']/*[local-name()='systemId']", doc, XPathConstants.NODE);
                if (systemId != null) {
                    systemId.setTextContent("ModifiedESP");
                    logger.debug("Updated systemId");
                };

                // Update transactionId
                Node transactionId = (Node) xpath.evaluate(
                        "//*[local-name()='responseId']/*[local-name()='transactionId']", doc, XPathConstants.NODE);
                if (transactionId != null)
                {
                    transactionId.setTextContent("ModifiedTxn123");
                    logger.debug("Updated transactionId");
                }

                ByteArrayOutputStream rawOutput= new ByteArrayOutputStream();
                Transformer transformer = TransformerFactory.newInstance().newTransformer();
                transformer.transform(new DOMSource(doc),new StreamResult(rawOutput));


                SaajSoapMessage soapMessage= (SaajSoapMessage) message;
                soapMessage.getSaajMessage().getSOAPPart().setContent(
                        new StreamSource(new ByteArrayInputStream(rawOutput.toByteArray()))
                );
                logger.info("SOAP response dynamically written from static XML template.");
            } catch (Exception e) {
                logger.error("Exception while processing business logic: {}", e.getMessage(), e);
                throw new SchemaValidationException("Error processing response XML", e);
            }
        }
    }

